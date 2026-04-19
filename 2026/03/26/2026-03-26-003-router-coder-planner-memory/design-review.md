# Design Review: RouterAgent Enhancement, Coder TaskAgent, Planner WorkflowAgent, Three-Level Memory

## Review Summary

The proposed design is well-structured and demonstrates strong understanding of the vage framework primitives. It correctly identifies that most capabilities (DAG execution, memory tiers, tool registries) already exist in vage and need only be wired at the vv application layer. However, several issues need correction based on careful analysis of the actual codebase.

---

## Issue 1: PlannerAgent Does Not Implement StreamAgent

**Severity:** High

**Problem:** The design proposes `PlannerAgent` as a `CustomAgent` (via `agent.RunFunc`). However, `CustomAgent` only implements `agent.Agent`, not `agent.StreamAgent`. The CLI's `selectAgent` method casts the routed agent to `agent.StreamAgent` and calls `RunStream`. If the planner is routed to, this cast will fail at runtime.

**Evidence:** `vage/agent/custom.go` line 34: `var _ Agent = (*CustomAgent)(nil)` -- only `Agent`, not `StreamAgent`. The CLI in `cli.go` line 105: `sa, ok := result.Agent.(agent.StreamAgent)`.

**Recommendation:** The `PlannerAgent` must implement `agent.StreamAgent`. The simplest approach: implement `RunStream` using `agent.RunToStream(ctx, a, req)`, which wraps any `Agent.Run` as a stream with lifecycle events. This is exactly how `workflowagent.Agent.RunStream` works (line 216 of `workflow.go`). Alternatively, make `PlannerAgent` a proper struct that embeds `agent.Base` and implements both `Run` and `RunStream` directly.

---

## Issue 2: CLI App Struct Only Holds Two Agents

**Severity:** High

**Problem:** The CLI `App` struct (in `cli/cli.go`) currently has fields `coder agent.StreamAgent` and `chat agent.StreamAgent`. The `New` function signature takes only these two. The design mentions "Update App struct to hold references to all five agents" in Task 9, but does not describe the `New` signature change needed, nor how the CLI handles the planner (which is not a `StreamAgent` out of the box).

**Evidence:** `cli/cli.go` lines 32-37 show the `App` struct; line 44 shows the `New` function signature.

**Recommendation:** The CLI does not actually need direct references to all five agents. The routing is handled by `routeFn` + `routes` which already generalizes to N agents. The `coder` and `chat` fields appear unused outside of `New` -- they are only passed via `routes`. The design should clarify that the `App.coder` and `App.chat` fields can be removed (or generalized to a map/slice) since routing is already agent-agnostic. The only change needed is updating the `routes` slice and `routeFn` fallback index.

---

## Issue 3: Tool Registry Helpers Return Wrong Type

**Severity:** Medium

**Problem:** The design proposes `RegisterReadOnly` and `RegisterReviewTools` returning `(*tool.Registry, error)`. But the existing `tools.Register` function returns `(*tool.Registry, error)`, while agents accept `tool.ToolRegistry` (interface). More importantly, the coder agent in `main.go` receives its registry wrapped by `vvcli.WrapRegistry` (for confirmation support). The researcher and reviewer agents should NOT be wrapped with confirmation since they only have read-only tools.

**Evidence:** `main.go` line 100: `reg := vvcli.WrapRegistry(toolRegistry, cfg.CLI.ConfirmTools)`. The coder gets the wrapped registry, but researcher/reviewer should get unwrapped registries.

**Recommendation:** `RegisterReadOnly` and `RegisterReviewTools` should return `(*tool.Registry, error)` (matching the existing pattern), and these registries should NOT be wrapped with `WrapRegistry`. Only the coder's full registry needs confirmation wrapping. The design should explicitly call this out.

---

## Issue 4: Planner Sub-Task Input Mapping Is Underspecified

**Severity:** Medium

**Problem:** When the planner builds DAG nodes from the plan, each node's `Runner` is a sub-agent. But the design does not specify how the sub-task description gets passed to the sub-agent. The `orchestrate.Node.InputMapper` field exists for this purpose but the design only shows `p.buildNodes(plan)` without explaining how each step's description becomes the sub-agent's input.

**Evidence:** `orchestrate/dag.go` `buildNodeInput` function (line 593) shows that without an `InputMapper`, nodes with no dependencies receive the original request, and nodes with dependencies receive upstream output messages. Neither case correctly passes the step description as the sub-agent's task.

**Recommendation:** Each node must set an `InputMapper` that creates a `RunRequest` with the step's description as the user message, plus any upstream results as context. Example:

```go
InputMapper: func(upstream map[string]*schema.RunResponse) (*schema.RunRequest, error) {
    msgs := []schema.Message{schema.NewUserMessage(step.Description)}
    // Optionally append upstream context
    for _, resp := range upstream {
        msgs = append(msgs, resp.Messages...)
    }
    return &schema.RunRequest{Messages: msgs, SessionID: req.SessionID}, nil
}
```

---

## Issue 5: FileStore Location Should Be in vv, Not vage

**Severity:** Medium

**Problem:** The design places `FileStore` in `vage/memory/filestore.go`, stating "The vage framework receives one new type (`memory.FileStore`)." However, the `FileStore` is application-specific filesystem persistence -- a concrete storage backend. The vage framework provides interfaces (`Store`) and in-memory implementations (`MapStore`). A filesystem-backed store is an application concern and belongs in vv.

**Evidence:** The requirement states "Persistent memory is stored as local files in a configurable directory" -- this is vv-specific behavior. The vage framework already provides `MapStore` as its concrete `Store`; `FileStore` is at the same abstraction level and should follow the same pattern of being provided by the application layer.

**Recommendation:** Place `FileStore` in `vv/memory/filestore.go` (new `vv/memory` package). This keeps vage as a pure framework without filesystem dependencies and follows the separation principle stated in the design's own architectural principles.

**Counter-argument considered:** Having `FileStore` in vage would make it reusable by other applications. However, the requirement explicitly scopes this to vv's local file storage, and the `memory.Store` interface already enables pluggable backends. If reuse is desired later, `FileStore` can be promoted to vage in a separate change.

---

## Issue 6: Session Memory Is Already Managed by CLI History

**Severity:** Medium

**Problem:** The CLI currently maintains conversation history in `App.history` (a `[]schema.Message` slice) and passes it to agents via `RunRequest.Messages`. The design proposes wiring `memory.SessionMemory` + `memory.Manager` into agents for session memory. However, the `taskagent.Agent.buildInitialMessages` loads session history from the memory manager AND accepts `req.Messages`. If both are populated, the agent will see duplicate messages -- once from session memory and once from the request messages.

**Evidence:** `taskagent/task.go` line 227: `buildInitialMessages` loads session messages first, then appends `reqMsgs`. In `cli.go` line 369, the CLI builds `req` with the full `m.app.history`.

**Recommendation:** Choose one approach, not both:

Option A (Recommended): Wire the memory manager and stop passing full history in `RunRequest.Messages`. Instead, the CLI passes only the latest user message in `req.Messages`, and the memory manager provides historical context. This is the cleaner separation and matches the memory architecture's intent.

Option B: Keep the CLI's current `history` array approach and do NOT wire session memory into agents. Use the memory manager only for persistent (store-tier) memory. This is simpler but loses the compression benefit.

The design should explicitly address this and pick one approach. Option A is recommended because it enables the sliding window compressor to manage context length, which is a key value-add of the memory system.

---

## Issue 7: HTTP Memory Endpoints File Location

**Severity:** Low

**Problem:** The design mentions placing HTTP memory handlers in `vv/cli/memory_handlers.go` as one option. HTTP handlers should not be in the `cli` package.

**Recommendation:** Place HTTP memory handlers in a new file `vv/httpapi/memory.go` or directly in `main.go` as closures (the design's second option). Given the current codebase has no `httpapi` package, inline handlers in `main.go` is the pragmatic choice for now.

---

## Issue 8: Planner's Plan Generation Should Use Streaming

**Severity:** Low

**Problem:** The plan generation phase uses `taskagent.Agent.Run` (non-streaming). For large, complex task descriptions, plan generation could take several seconds. Users see no feedback during this time.

**Recommendation:** Since the planner itself wraps the plan generation agent, and the planner's `RunStream` implementation would use `RunToStream`, the user only sees AgentStart/AgentEnd events. This is acceptable for the initial implementation. However, the design should note this as a known limitation and suggest that a future iteration could stream plan generation progress events.

---

## Issue 9: ConnectDAG Validation Rejects Disconnected Steps

**Severity:** Medium

**Problem:** The `orchestrate.ValidateDAG` function calls `checkConnected` which requires all nodes to form a single connected component (when treated as undirected). If the planner generates a plan with two independent sub-tasks that have no dependencies between them, the DAG validation will fail because those two nodes form disconnected components.

**Evidence:** `orchestrate/orchestrate.go` line 276: `checkConnected` returns an error if not all nodes are reachable from BFS starting at nodes[0].

**Recommendation:** For plans with independent steps, the planner must ensure the DAG is connected. The simplest approach: if the plan has multiple root nodes (no dependencies), add a synthetic "init" node that all roots depend on, or a synthetic "summary" node that depends on all terminals. Alternatively, use `orchestrate.RunDAG` directly instead of `workflowagent.NewDAG`, and the plan builder should add a final aggregation node. The design should specify this approach.

---

## Issue 10: Missing Error Return From Tool Registry Helpers

**Severity:** Low

**Problem:** The design's function signatures `RegisterReadOnly(cfg config.ToolsConfig) (*tool.Registry, error)` are correct but the functions must return errors from individual tool registrations. The existing `Register` function in `tools.go` shows the pattern -- each tool registration can fail and is wrapped with `fmt.Errorf`.

**Recommendation:** This is already correctly implied in the design. Just ensure the implementation follows the exact error-wrapping pattern from `tools.go`.

---

## Issue 11: Planner Fallback to Coder Has UX Gap

**Severity:** Low

**Problem:** The design states "If plan generation fails (invalid JSON, empty plan), fall back to routing the original request to the coder agent." However, the planner needs a reference to the coder agent for this fallback. The current design has `subAgents map[string]agent.Agent` which includes coder, so this works. But the fallback should also report that it fell back, so the user knows the planner could not decompose the task.

**Recommendation:** When falling back, the planner should log a warning and optionally prepend context to the coder's request indicating this is a fallback from planning failure.

---

## Issue 12: Router RunStream Does Not Stream Sub-Agent Output

**Severity:** Medium

**Problem:** The `routeragent.Agent.RunStream` uses `agent.RunToStream`, which wraps the entire `Run` call into AgentStart/AgentEnd events. This means when the router delegates to the coder (which supports real streaming), the coder's streaming events (TextDelta, ToolCallStart, etc.) are lost. The user only sees the final result.

**Evidence:** `routeragent/router.go` line 135: `return agent.RunToStream(ctx, a, req), nil`. This wraps `a.Run()` which calls `result.Agent.Run()` -- the non-streaming path.

**Recommendation:** This is an existing limitation, not introduced by this design. However, since the router is being enhanced to route to more agents, the impact is greater. The design should note this as a known limitation. The current CLI works around this because it calls `selectAgent` to get the routed agent and then calls `RunStream` on the selected agent directly (bypassing the router's `RunStream`). This pattern should be preserved for all new agents.

---

## Additional Suggestions

### S1: Batch Configuration Updates

The `MemoryConfig` defaults should be applied in the existing `applyDefaults` function in `config.go`, not as a separate step. This follows the established pattern.

### S2: Persistent Memory Context Injection Timing

The design says "At agent creation time, wrap the system prompt template to append persistent memory." This is incorrect -- persistent memory may change between sessions while agents are long-lived singletons. Instead, persistent memory should be loaded at session start and injected per-request. Use the `prompt.PromptTemplate` interface with a dynamic implementation that loads persistent memory entries on each `Render` call, or inject persistent memory content into the system prompt at the point of memory manager wiring (not at agent creation).

### S3: Planner MaxConcurrency Default

The design sets `MaxConcurrency: 2` for the DAG. This should be configurable via `AgentsConfig` or `MemoryConfig` (or a new `PlannerConfig`). Hardcoding to 2 is overly conservative for some workloads and too aggressive for others.

### S4: CLI Command Routing Order

The `/memory` command handling should be checked BEFORE `isExitCommand` in the `handleSubmit` method to avoid any potential conflicts. Actually, the existing code checks `isExitCommand` first, which is correct since `/memory` does not conflict with `/exit`. But the design should specify that `/memory` handling happens before agent routing (after exit check).
