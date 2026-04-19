## Dispatcher Refactoring: From Rigid Pipeline to Adaptive Decision Loop

### Background & Objectives

**Background:**
The current Orchestrator Agent (Dispatcher) in the `vv` module implements a fixed three-phase pipeline: explore -> classify -> dispatch. Every user request traverses the full pipeline regardless of complexity. The explorer agent always runs first to gather project context, the planner agent always runs second to classify and generate a plan, and only then does the dispatch phase execute. Once a plan (DAG) is generated, its topology is immutable -- steps execute in fixed order with no ability to adjust based on intermediate results.

This rigid architecture creates several problems:
1. **Wasted resources on simple tasks** -- A greeting like "hello" triggers a full explore + classify cycle before reaching the chat agent.
2. **Explorer and planner are fixed stages, not on-demand capabilities** -- Simple tasks do not need exploration; complex tasks may need multiple explorations at different points. The current architecture cannot adapt.
3. **Immutable plans** -- Once a DAG is built, it executes to completion without the ability to adjust remaining steps based on intermediate results, failures, or deviations.
4. **Two execution models** -- The Dispatcher uses a pipeline model while sub-agents (TaskAgent) use a ReAct loop, creating inconsistency in mental model and maintenance overhead.

**Objectives:**
- Unify the Dispatcher's execution model with the sub-agent model into a single adaptive decision loop: intent recognition -> plan tasks -> execute tasks (with dynamic replanning) -> summarize.
- Convert explorer and planner from mandatory fixed stages into on-demand capabilities that the LLM invokes when needed.
- Support dynamic replanning during task execution when steps fail or produce unexpected results.
- Introduce recursion depth control for nested agent invocations.
- Make the summarization phase optional and context-dependent.

### User Stories & Acceptance Criteria

**US-1: Unified Agent Main Loop**

As a developer using vv, I want the Orchestrator to use the same decision loop as sub-agents so that simple requests are handled immediately without unnecessary overhead.

Acceptance Criteria:
- A simple greeting (e.g., "hello") is routed directly to the chat agent without triggering exploration or planning.
- A coding request that requires project context triggers exploration on-demand before dispatching to the coder agent.
- A complex multi-step request triggers planning and produces a task list with dependencies.
- The Orchestrator follows a four-phase loop: intent recognition, planning, execution, summarization.
- Intent recognition and planning may be combined into a single LLM call for efficiency.

**US-2: On-Demand Explorer**

As a developer, I want the Orchestrator to explore project context only when the LLM determines it is needed, so that simple queries are not delayed by unnecessary exploration.

Acceptance Criteria:
- For requests that do not require project context (general questions, greetings), the explorer is not invoked.
- For requests that reference project files or code, the LLM triggers the explorer capability to gather context.
- The explorer can be invoked at any point during the decision loop, not only at the start.
- If the LLM determines additional context is needed mid-execution, it can invoke the explorer again.

**US-3: On-Demand Planner**

As a developer, I want the Orchestrator to generate execution plans only when the task requires multi-step coordination, so that single-agent tasks skip the planning overhead.

Acceptance Criteria:
- Tasks that can be handled by a single agent skip the planning phase entirely.
- Tasks requiring multiple agents or steps trigger the planner to generate a task list with dependencies.
- The planner capability can be invoked by the LLM as needed rather than being a mandatory pipeline stage.

**US-4: Dynamic Replanning During Execution**

As a developer, I want the Orchestrator to adjust remaining tasks when a step fails or produces unexpected results, so that the overall goal can still be achieved.

Acceptance Criteria:
- When a task step fails, the Orchestrator can trigger replanning to generate replacement steps for the remaining work.
- When a task step's output deviates significantly from expectations (as assessed by the LLM), replanning can be triggered.
- Replanning has a configurable maximum count (default: 2) to prevent infinite loops.
- When the maximum replan count is exceeded, execution stops and returns partial results with an error explanation.
- Normal task completions do not trigger replanning (replanning is not invoked on every step).

**US-5: Recursion Depth Control**

As a developer, I want nested agent invocations to be limited in depth, so that the system does not enter unbounded recursion when agents invoke sub-agents.

Acceptance Criteria:
- The Orchestrator (depth 0) runs the full decision loop: intent recognition, planning, execution, summarization.
- First-level sub-agents (depth 1) can plan but do not further decompose into sub-tasks.
- Agents at or beyond the maximum depth (default: 2) execute directly without planning.
- The current recursion depth is propagated through the execution context.

**US-6: Optional Summarization**

As a developer, I want the summarization phase to be configurable based on the interaction mode, so that CLI users are not shown redundant summaries while API consumers get complete responses.

Acceptance Criteria:
- In CLI streaming mode, summarization is skipped by default (the user has already seen the process output).
- In HTTP API mode, summarization is generated by default (the consumer may only see the final result).
- A configuration option allows forcing summarization on or off regardless of mode.
- Sub-agents generate summaries only when requested by their parent agent.

### Scope Boundaries

**In Scope:**
- Refactoring the Dispatcher's main execution flow (`vv/dispatches/dispatch.go`)
- Refactoring the classification/planning logic (`vv/dispatches/classify.go`) into a unified intent recognition + planning capability
- Refactoring the explorer from a fixed stage to an on-demand capability (`vv/dispatches/explore.go`)
- Adding dynamic replanning hooks to DAG execution (`vv/dispatches/dag.go`)
- Adding new type definitions for ReplanPolicy and SummaryPolicy (`vv/dispatches/types.go`)
- Adapting streaming event emission for the new phase model (`vv/dispatches/stream.go`)
- Adapting input construction and helper functions (`vv/dispatches/input.go`, `vv/dispatches/helpers.go`)
- Updating product design documents (orchestration process, models, dictionaries) to reflect the new architecture

**Out of Scope:**
- Changes to individual agent implementations (`vv/agents/*.go`) -- agents remain as-is; only their invocation patterns change
- Changes to the vage framework (`vage/agent/`, `vage/orchestrate/`) -- the refactoring is confined to the `vv/dispatches` package
- HTTP API response format changes -- the phase event structure may change, but the API contract (endpoints, request format) remains stable
- CLI TUI rendering logic -- the CLI adapts to whatever events are emitted; no CLI-specific changes needed beyond event handling
- Memory system changes
- Configuration file format changes (new options are additive)

### System Roles

| Role | Impact |
|------|--------|
| Orchestrator Agent | Major change: execution model changes from rigid pipeline to adaptive decision loop; gains on-demand explorer and planner capabilities; gains replanning and recursion depth awareness |
| Planner Agent | Invocation change: no longer a mandatory pipeline stage; invoked on-demand by the Orchestrator when multi-step planning is needed |
| Explorer Agent | Invocation change: no longer a mandatory first stage; invoked on-demand when project context is needed |
| Coder Agent | No change to implementation; may be invoked differently (directly for simple tasks without prior explore/classify) |
| Researcher Agent | No change |
| Reviewer Agent | No change |
| Chat Agent | No change |
| User | Improved experience: simple requests are faster; complex requests benefit from adaptive execution |
| Operator | New configuration options: summary policy, max replan count, max recursion depth |

### Models & States

**Existing models affected:**

1. **Task Plan** (`doc/prd/models/core/planner/model-task-plan.md`)
   - New attribute: `replan_count` (number) -- tracks how many times the plan has been replanned
   - New attribute: `max_replans` (number) -- maximum allowed replans (from ReplanPolicy)
   - New state transition: executing -> executing (on replan: remaining steps are replaced with newly generated steps)

2. **Plan Step** (`doc/prd/models/core/planner/model-plan-step.md`)
   - New attribute: `replan_generation` (number) -- which generation of the plan this step belongs to (0 = original, 1 = first replan, etc.)
   - Existing state "skipped" now also applies when a step is superseded by replanning

3. **Agent** (`doc/prd/models/core/agents/model-agent.md`)
   - New attribute: `summary_policy` (enum: Summary Policy) -- controls whether this agent generates execution summaries
   - New attribute: `max_recursion_depth` (number) -- maximum nesting depth for sub-agent invocations

**New models:**

4. **Replan Policy** (new, embedded in Task Plan configuration)
   - `trigger_on_failure` (boolean) -- trigger replanning when a step fails
   - `trigger_on_deviation` (boolean) -- trigger replanning when step output deviates from expectations
   - `max_replans` (number) -- maximum number of replans allowed (default: 2)

**New dictionaries:**

5. **Summary Policy** (new enum)
   - `auto` -- summarize in HTTP mode, skip in CLI mode
   - `always` -- always generate a summary
   - `never` -- never generate a summary

6. **Intent Type** (new enum)
   - `simple` -- task can be handled by a single agent directly
   - `complex` -- task requires multi-step planning and coordination

**Existing dictionaries affected:**

7. **Plan Status** (`doc/prd/dictionaries/core/dictionary-plan-status.md`)
   - No new values needed; "executing" state now also covers replanning (the plan remains in "executing" during replan)

8. **Plan Step Status** (`doc/prd/dictionaries/core/dictionary-plan-step-status.md`)
   - Existing "skipped" value now also covers steps superseded by replanning (clarify description)

### Business Processes & Rules

**Primary process affected: Orchestration** (`doc/prd/procedures/core/orchestration/procedure-orchestration.md`)

The orchestration process must be restructured from the current linear pipeline to an adaptive decision loop:

**Current flow:**
1. Receive instruction
2. Explore context (always)
3. Classify/plan (always)
4. Dispatch (direct or DAG)
5. Aggregate results
6. Respond

**New flow:**
1. Receive instruction
2. Intent recognition (may invoke explorer on-demand, may combine with planning)
3. Planning (on-demand, only for complex tasks; may invoke explorer for additional context)
4. Execution (sequential/parallel task execution with dynamic replanning on failure/deviation)
5. Summarization (optional, based on SummaryPolicy)
6. Respond

**New/updated business rules:**

| Rule ID | Rule Name | Description | Applicable Scenario |
|---------|-----------|-------------|---------------------|
| ORCH-25 | On-Demand Exploration | The explorer is invoked only when the LLM determines project context is needed, not as a fixed first stage | Intent recognition, Planning |
| ORCH-26 | On-Demand Planning | The planner is invoked only when the LLM determines the task requires multi-step coordination | Intent recognition |
| ORCH-27 | Intent-Plan Merge | For efficiency, intent recognition and planning may be combined into a single LLM call | Intent recognition + Planning |
| ORCH-28 | Simple Task Fast Path | When intent is simple and clear, execution proceeds directly to a single agent without planning | Intent recognition -> Execution |
| ORCH-29 | Replan on Failure | When a task step fails and ReplanPolicy.trigger_on_failure is true, the Orchestrator generates replacement steps for the remaining work | Execution |
| ORCH-30 | Replan on Deviation | When a task step's output deviates from expectations and ReplanPolicy.trigger_on_deviation is true, the Orchestrator generates replacement steps | Execution |
| ORCH-31 | Replan Limit | Replanning is limited by ReplanPolicy.max_replans (default: 2); exceeding the limit aborts execution with partial results | Execution |
| ORCH-32 | Recursion Depth Propagation | The current recursion depth is passed through the execution context; each sub-agent invocation increments the depth | Execution |
| ORCH-33 | Depth-Based Behavior | At depth 0 (Orchestrator): full decision loop. At depth 1: can plan but not decompose further. At depth >= maxDepth: direct execution only | All phases |
| ORCH-34 | Conditional Summarization | Summarization is controlled by SummaryPolicy: auto (HTTP=yes, CLI=no), always, or never | Summarization |
| ORCH-35 | Sub-Agent Summary Request | A parent agent may request its sub-agents to produce summaries via the execution context | Summarization |

**Updated rules:**

| Rule ID | Change | Description |
|---------|--------|-------------|
| ORCH-14 | Updated | Phase events change from fixed three phases (explore, plan, dispatch) to dynamic phases (intent, plan, execute, summarize) with phases appearing only when they actually occur |
| ORCH-21 | Updated | Streaming phase events are emitted only for phases that actually execute; phase count is determined dynamically |

### Applications & Pages

**CLI Application** (`doc/prd/applications/cli/`)
- Phase progress rendering must adapt to dynamic phase names and counts instead of fixed three phases
- No new pages required; existing main conversation view handles the new event types

**API Application** (`doc/prd/applications/api/`)
- Stream agent endpoint (`004-stream-agent.md`) emits different phase events (dynamic phases instead of fixed explore/plan/dispatch)
- Run agent endpoint (`003-run-agent.md`) response structure unchanged; summary may or may not be present based on policy
- No new endpoints required

### Non-functional Requirements

- **Performance Expectations**: Simple requests (greetings, direct questions) should reach the executing agent with at most one LLM call (intent recognition), compared to the current minimum of two LLM calls (explore + classify). Target: 50% or greater reduction in latency for simple tasks.
- **Data Scale**: Replanning generates additional plan steps. With max_replans=2, the worst case is 3x the original step count. This is bounded and acceptable.
- **Reliability**: The replan limit (default: 2) and recursion depth limit (default: 2) prevent unbounded resource consumption. Both limits are configurable by the Operator.
- **Backward Compatibility**: HTTP API endpoints remain unchanged. Phase event types in streaming responses will change (new phase names), which is a breaking change for clients that parse phase names. The change should be documented in release notes.
- **Testability**: Each concern is independently testable: intent recognition accuracy, on-demand explorer triggering, on-demand planner triggering, replan triggering conditions, recursion depth enforcement, summary policy behavior. Integration tests should cover: simple intent fast path, complex task planning, replan on failure, replan on deviation, replan limit exceeded, max depth reached, summary auto/always/never.
