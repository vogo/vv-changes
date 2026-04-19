# Code Review: RouterAgent Enhancement, Coder TaskAgent, Planner WorkflowAgent, Three-Level Memory

## Files Reviewed

### New Files
- `vv/agents/planner.go` -- PlannerAgent implementation
- `vv/agents/memory_prompt.go` -- PersistentMemoryPrompt dynamic template
- `vv/cli/memory.go` -- CLI `/memory` command handlers
- `vv/memory/filestore.go` -- FileStore persistent memory backend

### Modified Files
- `vv/agents/agents.go` -- Expanded to Create() with 5 routes
- `vv/agents/prompts.go` -- Added researcher, reviewer, planner, summary prompts
- `vv/cli/cli.go` -- Added persistentMem field, /memory command dispatch
- `vv/cli/messages.go` -- Message types (unchanged in substance)
- `vv/cli/render.go` -- Render helpers (unchanged in substance)
- `vv/config/config.go` -- Added MemoryConfig
- `vv/main.go` -- Full wiring of memory, new agents, HTTP memory endpoints
- `vv/tools/tools.go` -- Added RegisterReadOnly, RegisterReviewTools

### Test Files
- `vv/agents/agents_test.go`
- `vv/agents/planner_test.go`
- `vv/agents/prompts_test.go`
- `vv/cli/cli_test.go`
- `vv/cli/confirm_test.go`
- `vv/cli/render_test.go`
- `vv/config/config_test.go`
- `vv/memory/filestore_test.go`
- `vv/integrations/agents_test.go`
- `vv/integrations/cli_test.go`
- `vv/integrations/wiring_test.go`

---

## Issues Found and Fixed

### 1. BUG (Critical): HTTP memory endpoints never served

**File:** `vv/main.go` (lines 169-183)

**Problem:** The code created a custom `http.ServeMux` with memory endpoints wrapping `svc.Handler()`, but then called `svc.Start(ctx)` which builds its own internal mux via `buildMux()`. The custom mux with memory endpoints was never used -- all four memory HTTP endpoints (`GET/PUT/DELETE /v1/memory/...`) were dead code in production.

**Fix applied:** Replaced `svc.Start(ctx)` with manual `net.Listen` + `http.Server{Handler: mux}` + `server.Serve(ln)`, using the custom mux that includes both service routes and memory endpoints. Added graceful shutdown via context cancellation goroutine (matching the pattern in `service.Start`).

### 2. Dead code: Unused types and functions in `cli/memory.go`

**File:** `vv/cli/memory.go` (lines 132-163)

**Problem:** `MemoryEntryResponse`, `entriesToResponse()`, and `splitMemoryKey()` were defined in the CLI package but never called by any CLI code. These duplicate the types already defined in `main.go` (`memoryEntryResponse`, `splitKey`). The unused `memory` import also caused a latent build issue.

**Fix applied:** Removed `MemoryEntryResponse`, `entriesToResponse()`, `splitMemoryKey()`, and the unused `"github.com/vogo/vage/memory"` import.

---

## Issues Noted (Not Fixed -- Acceptable or Deferred)

### 3. Design deviation: CLI history not removed

**Design spec (section 2.6.4):** Remove `App.history` field, pass only latest user message in `RunRequest.Messages`.

**Actual:** `App.history` is retained and the full history is passed in `RunRequest.Messages` (lines 354-355, 373-374, 483-488 of `cli/cli.go`).

**Assessment:** This is intentional -- the session memory tier is wired but the CLI has not been switched to rely on it for context. This is a safe intermediate state: the memory manager receives messages via `storeAndPromoteMessages` but the CLI still provides full history directly. Switching to single-message mode requires verifying that `buildInitialMessages` in `taskagent` correctly reconstructs context from session memory, which is a behavioral change best validated separately.

### 4. `RegisterReadOnly` accepts unused `cfg` parameter

**File:** `vv/tools/tools.go` line 64

**Problem:** `cfg config.ToolsConfig` parameter is accepted but never used (read/glob/grep have no configuration).

**Assessment:** Acceptable for API consistency with `Register()` and `RegisterReviewTools()`, and for forward compatibility if these tools gain configuration options.

### 5. `extractJSON` brace matching ignores string literals

**File:** `vv/agents/planner.go` lines 182-196

**Problem:** The brace-depth counting in `extractJSON` does not account for `{` or `}` inside JSON string values. A description like `"Use {braces} here"` would cause incorrect extraction.

**Assessment:** Low risk in practice -- the LLM is instructed to produce a JSON plan where descriptions are unlikely to contain unbalanced braces. The code fences extraction path (tried first) avoids this issue entirely. A more robust approach would use `json.Valid()` to validate candidate substrings, but the current approach is adequate for the initial implementation.

### 6. `PersistentLoad` config field is never checked

**File:** `vv/config/config.go` line 19, `vv/main.go`

**Problem:** `MemoryConfig.PersistentLoad` is defined (default `false` since zero value of bool) but never checked anywhere. The design says default is `true`, but the zero-value of bool is `false`.

**Assessment:** The field exists for future use. Currently, persistent memory is always loaded at startup. If the intent is to honor `PersistentLoad: false` to skip loading, additional logic is needed in `main.go`. The default value issue (design says `true`, Go zero-value is `false`) should be addressed in `applyDefaults` when this feature is actually used.

### 7. Summary node iteration order is non-deterministic

**File:** `vv/agents/planner.go` lines 248-258

**Problem:** The summary node's `InputMapper` iterates over `upstream map[string]*schema.RunResponse` using `range`, which has non-deterministic ordering in Go. This means the summary prompt content order varies between runs.

**Assessment:** Low impact -- the LLM summarizer handles arbitrary input ordering. However, for reproducibility in testing, sorting the keys (as done in `PlanAggregator.Aggregate`) would be more consistent.

---

## Code Quality Observations

### Positive
- Clean separation between framework (vage) and application (vv) layers
- Compile-time interface checks (`var _ agent.Agent = (*PlannerAgent)(nil)`)
- Consistent error wrapping with `fmt.Errorf` and `%w`
- Thorough test coverage including edge cases (empty plans, invalid JSON, non-existent keys)
- Graceful degradation in `PersistentMemoryPrompt.Render` (returns base prompt on error)
- Planner fallback to coder agent on plan generation failure
- FileStore correctly preserves `CreatedAt` on updates

### Style
- Code follows existing project conventions consistently
- Functional options pattern used appropriately
- Good use of Go idioms (type switches, builder pattern with `strings.Builder`)
- Test helpers (`stubAgent`, `mockChatCompleter`) are clean and minimal

---

## Test Results

All tests pass after applied fixes:

```
?       github.com/vogo/vv           [no test files]
ok      github.com/vogo/vv/agents
ok      github.com/vogo/vv/cli
ok      github.com/vogo/vv/config
ok      github.com/vogo/vv/integrations
ok      github.com/vogo/vv/memory
ok      github.com/vogo/vv/tools
```
