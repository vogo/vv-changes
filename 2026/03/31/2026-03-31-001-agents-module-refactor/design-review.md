# Design Review: vv/agents Module Refactor

## Review Summary

The proposed design is well-structured and addresses the key pain points identified in the requirements. The decomposition into `registry/`, `dispatch/`, `agents/`, `lifecycle/`, and `setup/` is sound. Below are specific improvements organized by severity and package.

---

## Issue 1: `ToolProfile.BuildRegistry` Cannot Filter From a Full Registry

**Severity:** High -- blocks implementation

**Problem:** The design proposes `ToolProfile.BuildRegistry(fullReg *tool.Registry)` that "filters tools from a complete registry based on capabilities." However, looking at the actual `tools/tools.go` code, the `tool.Registry` is populated by calling individual `Register()` functions (e.g., `bashtool.Register(reg)`) -- there is no public API on `tool.Registry` to enumerate registered tools and selectively copy them to a new registries. The `vage` framework does not expose a `Filter()` or `List()` method on `tool.Registry`.

**Current tool registration approach:** Each tool type has its own `Register(reg)` function that adds the tool to a registries. The three registries in `main.go` (`Register`, `RegisterReadOnly`, `RegisterReviewTools`) each call these functions independently with the appropriate tool subset.

**Suggestion:** `BuildRegistry` should NOT attempt to filter an existing registries. Instead, it should construct a new `tool.Registry` from scratch by calling the individual tool registration functions based on capabilities. This means `BuildRegistry` needs access to `config.ToolsConfig` (for bash timeout, working dir) rather than a `*tool.Registry`. Alternatively, `BuildRegistry` can accept a `ToolsConfig` parameter and call the appropriate `xxxtools.Register()` functions directly.

**Revised approach:**
```go
func (p ToolProfile) BuildRegistry(cfg config.ToolsConfig) (*tool.Registry, error) {
    reg := tool.NewRegistry()
    for _, cap := range p.Capabilities {
        if err := registerCapabilityTools(reg, cap, cfg); err != nil {
            return nil, err
        }
    }
    return reg, nil
}
```

This eliminates the assumption that `tool.Registry` supports filtering, and aligns with how tools are actually registered in the codebase.

**Impact:** Changes `BuildRegistry` signature, `setup.New()` signature (no longer passes `fullToolReg`), and the `Dispatcher.fullToolReg` field.

---

## Issue 2: `setup.New()` Silently Ignores Errors for Explorer/Planner

**Severity:** High -- silent failures in production

**Problem:** In the proposed `setup.go`, the explorer and planner creation uses the blank identifier for errors:
```go
explorerDesc, _ := reg.Get("explorer")
explorerToolReg, _ := explorerDesc.ToolProfile.BuildRegistry(fullToolReg)
explorer, _ := explorerDesc.Factory(...)
```

These are infrastructure agents critical to the orchestration flow. If registration is missing or factory creation fails, the dispatcher receives nil agents and will panic at runtime.

**Suggestion:** All errors must be checked and returned. Add explicit error handling for non-dispatchable agent construction.

---

## Issue 3: `PersistentMemoryPrompt` Integration Missing from Factory

**Severity:** High -- behavioral regression

**Problem:** The current `newCoderAgent()` accepts a `persistentPrompt prompt.PromptTemplate` parameter and uses it to replace the base system prompt when available (see `coder.go` lines 38-41). The design mentions this in passing ("The `RegisterCoder` factory checks if a persistent memory store is available") but the `RegisterCoder` example code uses `prompt.StringPrompt(CoderSystemPrompt)` unconditionally. The `FactoryOptions` struct lists a `PersistentMemory` field but this is not shown as part of the struct definition in section 2.1.

**Suggestion:** Add `PersistentMemory memory.Memory` to `FactoryOptions` struct explicitly. In the `RegisterCoder` factory, construct a `PersistentMemoryPrompt` when `opts.PersistentMemory != nil` and use it in place of the static `StringPrompt`. Show this clearly in the code example to prevent implementors from missing it.

---

## Issue 4: CLI Confirmation Wrapping Not Addressed

**Severity:** Medium -- behavioral regression risk

**Problem:** In current `main.go`, the coder's tool registry is wrapped with CLI confirmation: `coderReg := vvcli.WrapRegistry(toolRegistry, cfg.CLI.ConfirmTools)`. The design mentions this in a comment ("This is done by the caller (main.go) before passing fullToolReg") but the `ToolProfile.BuildRegistry()` approach creates new registries from scratch, so wrapping the `fullToolReg` does NOT propagate confirmation to the coder's profile-built registries.

**Suggestion:** The CLI confirmation wrapping must happen after `BuildRegistry` creates the coder's registry, not before. Add a post-creation hook in `setup.New()` or accept a `ToolRegistryWrapper` function in the setup options. The simplest approach: `setup.New()` accepts an optional `WrapToolRegistry func(*tool.Registry) *tool.Registry` parameter that is applied to dispatchable agents' registries (or just the coder's, matching current behavior).

---

## Issue 5: `dispatch` Package Name Conflicts with Built-in Concepts

**Severity:** Low -- naming clarity

**Problem:** `dispatch` is a reasonable name but `classify.go` inside it is described as containing `explore()` and `exploreStream()` methods. These are exploration methods, not classification methods. The file name `classify.go` does not accurately describe its contents.

**Suggestion:** Rename `classify.go` to `routing.go` or split into `explore.go` (explore/exploreStream) and `classify.go` (classify/classifyStream/classifyDirect). Since the file currently mixes two concerns (exploration and classification), splitting improves readability.

---

## Issue 6: `DynamicAgentSpec.ToolAccess` Kept as String for JSON Compat

**Severity:** Medium -- incomplete migration

**Problem:** The design keeps `DynamicAgentSpec.ToolAccess` as a `string` type for JSON compatibility, but this means the old `ToolAccessLevel` string constants ("full", "read-only", "none") must still be supported. The design does not specify how `validate()` maps these strings to `ToolProfile` values.

**Suggestion:** Add a `ProfileByName(name string) (ToolProfile, bool)` function to the `registry` package that maps profile name strings to `ToolProfile` values. The `buildDynamicAgent` method should use this function to resolve `spec.ToolAccess` string to a `ToolProfile`. Document the mapping explicitly.

---

## Issue 7: `Dispatcher` Options Struct is Over-Broad

**Severity:** Medium -- maintainability

**Problem:** The `Options` struct has 13 fields, including both the `Registry` and pre-built `SubAgents` map. This is redundant -- if the registry contains factories and the setup layer builds agents from it, the dispatcher should either receive the registry and build agents itself, or receive pre-built agents without needing the registries.

**Suggestion:** The `Dispatcher` needs the registry only for `ClassifyResult.validate()` and `buildDynamicAgent()`. The pre-built `SubAgents` map is needed for runtime dispatch. Both are legitimate needs. However, `PlanGen` (the plan summarizer) should be created inside setup and passed as `SubAgents["plan-gen"]` or as a dedicated field -- not as a `*taskagent.Agent`. Use `agent.Agent` interface type for consistency.

Also, separate the `Options` into required fields (constructor params) and optional fields (functional options pattern) to match the codebase convention of `taskagent.WithXxx()`:

```go
func New(reg *registries.Registry, subAgents map[string]agent.Agent, opts ...Option) *Dispatcher
```

---

## Issue 8: Hook Integration Point Too Narrow

**Severity:** Low -- design gap

**Problem:** The design shows `runWithHooks` wrapping `runDirect` and `forwardSubAgentStream`, but `runPlan` dispatches via `orchestrate.ExecuteDAG` which runs sub-agents internally through `orchestrate.Node.Runner`. Hooks would not be invoked for DAG-executed agents unless the node runners are wrapped.

**Suggestion:** To apply hooks to DAG-executed agents, wrap each `orchestrate.Node.Runner` with a hook-aware wrapper at `buildNodes` time:

```go
func (d *Dispatcher) wrapWithHooks(agentID string, runner agent.Agent) agent.Agent {
    return &hookedAgent{inner: runner, hooks: d.hooks, agentID: agentID}
}
```

This ensures hooks fire for both direct dispatch and DAG execution without modifying the `vage` orchestrate framework.

---

## Issue 9: `setup.Result` Exposes Internal Wiring

**Severity:** Low -- API hygiene

**Problem:** `setup.Result` exposes `Registry` and raw `SubAgents` map. The HTTP service in `main.go` needs specific agents for registration, but exposing the full registry and internal map is broader than needed.

**Suggestion:** Instead of exposing `SubAgents map[string]agent.Agent`, provide a method `Agents() []agent.Agent` that returns the agents suitable for HTTP registration. The `Registry` field can be kept but documented as optional (for advanced use cases like dynamic agent introspection).

---

## Issue 10: Missing Error Return from `Registry.Register`

**Severity:** Low -- design choice

**Problem:** `Register` panics on duplicate ID. While this is a startup-time programming error, panics make testing harder and are inconsistent with the Go convention of returning errors from public APIs.

**Suggestion:** Keep the panic approach but add a `MustRegister` name to make the panic behavior explicit:

```go
func (r *Registry) MustRegister(d AgentDescriptor) // panics on duplicate
func (r *Registry) Register(d AgentDescriptor) error // returns error on duplicate
```

Use `MustRegister` in `setup.go` where panicking is acceptable.

---

## Issue 11: `PlanSummaryPrompt` Location

**Severity:** Low -- package cohesion

**Problem:** The design moves `PlanSummaryPrompt` to `dispatch/` package. However, it is used by `setup.go` to construct the plan summarizer agent. This creates a dependency from `setup/` to `dispatch/` just for a string constant.

**Suggestion:** Keep `PlanSummaryPrompt` in `dispatch/` since it is semantically part of the dispatch flow. The `setup/` -> `dispatch/` dependency already exists for the `dispatch.New()` call, so this adds no new dependency edge. No change needed -- this is fine as proposed.

---

## Issue 12: `capabilityToolNames` Hardcoding vs. Tool Registration Names

**Severity:** Medium -- fragility

**Problem:** The `capabilityToolNames` map hardcodes tool names like `"file_read"`, `"file_write"`, `"file_edit"`, `"bash"`, `"glob"`, `"grep"`. However, looking at `tools/tools.go`, the actual tool names come from each tool's `Register()` function (e.g., `readtool.Register`, `bashtool.Register`). If the `vage` framework changes a tool name, the hardcoded map breaks silently.

**Suggestion:** Since we are already suggesting Issue 1's approach (calling tool registration functions directly in `BuildRegistry`), this map becomes unnecessary. The capability-to-tool mapping is encoded procedurally in the `registerCapabilityTools` function. Remove `capabilityToolNames` entirely.

---

## Summary of Changes

| Issue | Severity | Action |
|-------|----------|--------|
| 1. BuildRegistry cannot filter | High | Use `ToolsConfig` to construct registries from scratch |
| 2. Silent error ignoring in setup | High | Add proper error handling |
| 3. PersistentMemoryPrompt missing | High | Add to FactoryOptions, show in coder example |
| 4. CLI confirmation wrapping | Medium | Add WrapToolRegistry option to setup |
| 5. classify.go naming | Low | Split into explore.go and classify.go |
| 6. DynamicAgentSpec string mapping | Medium | Add ProfileByName() function |
| 7. Options struct too broad | Medium | Use functional options for Dispatcher |
| 8. Hook integration for DAG | Low | Wrap node runners with hooks |
| 9. Result exposes internals | Low | Add Agents() method |
| 10. Register panic vs error | Low | Add MustRegister naming |
| 11. PlanSummaryPrompt location | Low | No change needed |
| 12. capabilityToolNames fragility | Medium | Remove in favor of procedural registration |
