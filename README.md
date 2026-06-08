# a2bench

A2A-based universal adapter for agent benchmarks — one interface, all harnesses.

## Problem

Every agent benchmark (SWE-bench, WebArena, GAIA, Terminal-Bench, OSWorld, Tau-Bench, etc.) defines its own harness interface. Building an agent means adapting it to each benchmark's specific protocol (bash CLI, tool-calling API, MCP, conversational, code generation). This is O(agents × benchmarks) integration work.

## Idea

Use the A2A protocol as the universal agent interface. Build adapters from A2A to each benchmark harness. An agent that speaks A2A can be evaluated on any benchmark through the appropriate adapter.

```
Agent (speaks A2A) → a2bench adapter → SWE-bench harness
                   → a2bench adapter → WebArena harness
                   → a2bench adapter → GAIA harness
                   → a2bench adapter → Terminal-Bench harness
```

## Status

Research phase. See RESEARCH.md for landscape analysis.
