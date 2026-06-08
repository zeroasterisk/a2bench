# Porting Exgentic to A2A: Detailed Feasibility Analysis

## 1. Executive Summary

Exgentic's Unified Protocol and A2A solve the same problem (decouple agents from benchmarks) with different approaches. Exgentic's protocol is a Python-native, in-process mediation layer. A2A is a network protocol (JSON-RPC/gRPC/REST) with standard types. Porting is feasible and would make Exgentic the first benchmark harness that can evaluate any A2A-speaking agent over the network, not just Python agents in the same process.

**Effort estimate:** Medium. The core mapping is clean. The work is mostly in the agent-side adapter (replacing the in-process AgentInstance with an A2A client) and adding an A2A server for benchmark exposure.

---

## 2. Architecture Comparison

### Exgentic Today
```
Orchestrator
  ├── BenchmarkSession (in-process or runner-isolated)
  │     ├── start() → Observation
  │     ├── step(Action) → Observation
  │     └── score() → SessionScore
  └── AgentInstance (in-process or runner-isolated)
        ├── start(task, context, actions)
        └── react(Observation) → Action
```

### With A2A
```
Orchestrator
  ├── BenchmarkSession (unchanged)
  │     ├── start() → Observation
  │     ├── step(Action) → Observation
  │     └── score() → SessionScore
  └── A2AAgentAdapter (NEW — replaces AgentInstance)
        ├── Wraps benchmark as A2A task
        ├── Sends message/send to agent's A2A endpoint
        ├── Receives agent's response (tool calls / messages)
        └── Translates back to Exgentic Action
```

---

## 3. Protocol Mapping: Exgentic ↔ A2A

### 3.1 Session Lifecycle

| Exgentic | A2A | Mapping |
|---|---|---|
| `Session.start()` → first Observation | `message/send` with task description | Task description + context → A2A Message parts |
| `AgentInstance.react(obs)` → Action | Agent processes message, returns response | A2A response contains tool calls or text |
| `Session.step(action)` → Observation | Execute action, send result back as follow-up | Tool results → A2A Message parts on same task |
| `Session.done()` / agent returns None | Task reaches terminal state | A2A task state: completed/failed |
| `Session.score()` | (post-task evaluation, outside A2A) | Benchmark evaluates after A2A task completes |

### 3.2 Data Structure Mapping

**Task/Context/Actions → A2A Agent Card + First Message**

```
Exgentic                          A2A
────────                          ───
task: str                    →    First message text part
context: Dict[str, Any]     →    Message metadata or structured data part
actions: List[ActionType]   →    Agent Card skills (discovery) OR
                                  message context describing available tools
```

**ActionType → A2A has no direct equivalent**

This is the key gap. Exgentic's `ActionType` is a rich schema:
```python
ActionType(name, description, cls: type[BaseModel], is_message, is_finish, is_hidden)
```

A2A doesn't define tool schemas — it defines task/message/artifact types. Options:
1. **Encode action schemas in the initial message** as structured data (JSON schema in a data part)
2. **Use MCP for tool exposure** alongside A2A for task management (A2A for task lifecycle, MCP for tool calling)
3. **Map actions to A2A skills** in the agent card (limited — skills are descriptive, not executable)

**Recommendation: Option 2** — Use A2A for the task lifecycle and message exchange. Use MCP for the tool/action protocol. This matches how real agents work: they speak A2A for task coordination and MCP for tool access.

**Action → A2A Message (agent response)**

```
Exgentic                          A2A
────────                          ───
SingleAction(name, arguments) →   Message with structured data part
                                  containing {tool: name, args: {...}}
MessageAction(content)        →   Message with text part
FinishAction                  →   Task state → completed
ParallelAction                →   Message with multiple structured parts
```

**Observation → A2A Message (benchmark response)**

```
Exgentic                          A2A
────────                          ───
SingleObservation(result)     →   Message with text/data part (tool result)
MultiObservation              →   Message with multiple parts
MessageObservation            →   Message with text part (user message)
EmptyObservation              →   No-op / acknowledgment message
```

### 3.3 Session State Mapping

```
Exgentic SessionStatus         A2A TaskState
──────────────────            ─────────────
(session created)          →  submitted
(agent.start() called)    →  working
(loop running)             →  working
(agent returns None)       →  completed
(benchmark done)           →  completed
(limit reached)            →  failed
(error)                    →  failed
(agent needs input)        →  input-required
```

---

## 4. Implementation Plan

### 4.1 New A2A Agent Adapter (core change)

Create `exgentic/adapters/agents/a2a_agent.py`:

```python
class A2AAgentInstance(AgentInstance):
    """Wraps an external A2A-speaking agent."""
    
    def __init__(self, agent_url: str, session_id: str):
        self.client = A2AClient(agent_url)  # from a2a-python SDK
        self.task_id = None
        self.context_id = session_id
    
    def start(self, task, context, actions):
        # Build initial A2A message with task + context + action schemas
        message = self._build_initial_message(task, context, actions)
        # Send to agent, get task ID back
        result = self.client.send_message(message, context_id=self.context_id)
        self.task_id = result.id
    
    def react(self, observation: Observation) -> Optional[Action]:
        # Translate observation to A2A message
        message = self._observation_to_message(observation)
        # Send as follow-up on existing task
        result = self.client.send_message(message, task_id=self.task_id)
        # Translate A2A response back to Exgentic Action
        return self._response_to_action(result)
    
    def _build_initial_message(self, task, context, actions):
        parts = [
            TextPart(text=task),
            DataPart(data={"context": context, "available_actions": [
                {"name": a.name, "description": a.description, 
                 "parameters": a.arguments.model_json_schema()}
                for a in actions
            ]})
        ]
        return Message(role="user", parts=parts)
    
    def _observation_to_message(self, obs):
        # Tool result → A2A message
        parts = [TextPart(text=str(obs.result))]
        return Message(role="user", parts=parts)
    
    def _response_to_action(self, result):
        # Parse agent's response for tool calls or messages
        # This is where the translation complexity lives
        ...
```

### 4.2 A2A Agent Configuration

Create `exgentic/agents/a2a/`:

```python
class A2AAgent(Agent):
    display_name = "A2A Agent"
    slug_name = "a2a"
    
    agent_url: str  # A2A endpoint URL
    agent_card_url: str | None = None  # Optional agent card for discovery
    
    def _get_instance_class(self):
        return A2AAgentInstance
```

### 4.3 A2A Benchmark Server (optional, for exposing benchmarks)

Create `exgentic/adapters/benchmark_server/a2a_server.py`:

This would expose an Exgentic benchmark as an A2A endpoint, allowing external orchestrators (not just Exgentic) to run benchmark tasks:

```python
class BenchmarkA2AServer:
    """Exposes an Exgentic benchmark as an A2A agent endpoint."""
    
    def __init__(self, benchmark: Benchmark):
        self.benchmark = benchmark
        self.evaluator = benchmark.get_evaluator()
        self.active_sessions: dict[str, Session] = {}
    
    async def handle_message(self, task_id, message):
        # On first message: create session, return task description
        # On subsequent: execute action from message, return observation
        ...
```

### 4.4 Registration

Add to `interfaces/registry.py`:
```python
AGENTS["a2a"] = RegistryEntry(module="exgentic.agents.a2a", class_name="A2AAgent")
```

### 4.5 Runner Considerations

The A2A agent adapter doesn't need Exgentic's runner system (thread/process/docker/venv) because the agent is already running externally. The adapter is always `direct` — it's just an HTTP client.

---

## 5. Challenges & Gaps

### 5.1 Action Schema Translation (HARD)

Exgentic's `ActionType` carries a Pydantic model for argument validation. A2A messages are untyped text/data parts. The agent needs to understand the tool schemas and return properly structured responses.

**Options:**
- Embed JSON Schema in the initial message (works for any LLM agent)
- Use MCP alongside A2A (cleaner for tool-calling agents)
- Define a convention: agent returns JSON with `{tool: name, args: {...}}` in a data part

### 5.2 Synchronous vs Async (MEDIUM)

Exgentic's loop is synchronous: `react()` → `step()` → `react()` → ...
A2A supports both blocking (`message/send` with blocking=true) and async patterns.

For benchmarks, blocking mode is fine — we need the response before proceeding. But streaming would be better for observability.

### 5.3 Multi-Turn Conversation Mapping (MEDIUM)

Exgentic's loop is: observation → action → observation → action → ...
A2A's model is: message → response (potentially with follow-ups via taskId).

These map well: each Exgentic turn = one A2A follow-up message on the same task. The A2A `contextId` = Exgentic `session_id`. The A2A `taskId` can be reused across turns.

### 5.4 Finish Detection (EASY)

Exgentic has explicit `FinishAction`. In A2A, the agent sets task state to `completed`. The adapter watches for terminal task states.

### 5.5 Parallel Actions (MEDIUM)

Exgentic supports `ParallelAction` (multiple actions in one turn). A2A messages can contain multiple parts, but there's no explicit parallel execution semantic. Would need to encode as structured data.

### 5.6 Score/Evaluation (NOT AN A2A CONCERN)

Scoring is benchmark-internal — it happens after the session ends, outside the A2A protocol. No mapping needed.

---

## 6. What Changes in Exgentic vs What Stays

### Stays Unchanged
- All benchmark implementations (swebench, tau2, appworld, etc.)
- Session interface and lifecycle
- Evaluator interface
- Orchestrator core loop
- All existing agent implementations (they still work)
- Observer/controller system
- Registry, CLI, dashboard
- Runner system (for non-A2A agents/benchmarks)

### New Code
- `adapters/agents/a2a_agent.py` — A2A agent adapter (~200 lines)
- `agents/a2a/` — A2A agent config + registration (~50 lines)
- `adapters/benchmark_server/a2a_server.py` — Optional benchmark-as-A2A-endpoint (~300 lines)

### Modified Code
- `interfaces/registry.py` — Add A2A agent to registry (2 lines)
- `pyproject.toml` — Add a2a-python SDK dependency

### Estimated Total: ~550 lines of new code, <10 lines modified

---

## 7. Comparison: Fork+Extend vs Build From Scratch

| Factor | Fork Exgentic | Build from Scratch |
|--------|--------------|-------------------|
| Benchmark integrations | 7 benchmarks ready | Must rewrite each |
| Agent integrations | 7 agents + A2A | Just A2A |
| Orchestration | Proven, tested | Must build |
| Observability | Observers, dashboard, OTel | Must build |
| Runner isolation | 6 runner types | Not needed (A2A is network) |
| Cost tracking | LiteLLM integration | Must build |
| Community/maintenance | Active project | Solo maintenance |
| Complexity | Large codebase | Minimal |
| A2A-native design | Bolted on | First-class |

**Recommendation: Fork Exgentic, add A2A adapter.** The 7 benchmark integrations alone would take weeks to rewrite. Adding an A2A agent adapter is ~550 lines. The existing orchestrator, observers, and scoring are production-tested.

---

## 8. Stretch Goal: A2A as the Wire Protocol

Beyond just adding an A2A agent adapter, Exgentic's entire orchestrator-to-agent communication could be rewritten to use A2A as the wire protocol (replacing the custom Transport/Runner system). This would mean:

- Agent isolation via network (A2A endpoint) instead of process/docker runners
- Benchmark exposure via A2A server (any orchestrator can run benchmarks)
- Full protocol observability via A2A's streaming/push notifications

This is a larger refactor but would make Exgentic the reference implementation of "benchmarks as A2A services."

---

## 9. Next Steps

1. **Fork Exgentic** to zeroasterisk org
2. **Implement A2AAgentInstance** adapter
3. **Test with a simple A2A agent** (e.g., Scion agent via scion-a2a-bridge)
4. **Run GSM8k benchmark** as proof-of-concept (simplest benchmark)
5. **Iterate** to SWE-bench (most important benchmark)
6. **Optionally** build benchmark-as-A2A-server for external consumption
