# Requirements: RouterAgent, Coder TaskAgent, Planner WorkflowAgent, Three-Level Memory

## 1. RouterAgent as the Front Door

All user input enters a RouterAgent that classifies intent and dispatches to the right sub-agent (coder, planner, researcher, reviewer, or chat).

## 2. Coder as the Primary TaskAgent

A ReAct-loop agent with all file tools (read, write, edit, glob, grep, bash). This is the workhorse.

## 3. Planner Uses WorkflowAgent

For complex multi-step tasks, the planner decomposes into a DAG and orchestrates via vage's orchestrate engine.

## 4. Three-Level Memory

- Working memory per request
- Session memory per conversation
- Persistent store for long-term knowledge
