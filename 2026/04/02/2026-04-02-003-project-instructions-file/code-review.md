# Code Review: Project Instructions File (VV.md)

## Summary

The implementation correctly follows the design document. Project instructions are loaded from `VV.md`, flowed through `Config` -> `FactoryOptions` -> agent factories and dispatcher, and appended to system prompts. The code is clean, well-structured, and consistent with the codebase conventions.

## Issues Found

### Issue 1: Missing project instructions in `reassessIntent` (Bug)

**Severity**: Medium  
**File**: `vv/dispatches/intent.go`, function `reassessIntent()` (line 175)

The `reassessIntent` method constructs a system prompt from `d.intentSystemPrompt` but does **not** append project instructions. This method is called after exploration to re-classify the intent. Since the initial `recognizeIntentDirect` call includes project instructions, the re-assessment should also include them for consistency. Without this, the re-assessment LLM call operates without project context, which could lead to different routing decisions.

**Fix**: Add `systemPrompt = appendProjectInstructions(systemPrompt, d.projectInstructions)` after the fallback assignment and before the `{{.WorkingDir}}` substitution.

### Issue 2: Missing project instructions in `classifyDirect` (Bug)

**Severity**: Medium  
**File**: `vv/dispatches/intent.go`, function `classifyDirect()` (line 635)

The `classifyDirect` method constructs a system prompt but does not append project instructions. This is the fallback classification path when no planner agent is configured. It should include project instructions for consistency with `recognizeIntentDirect`.

**Fix**: Add `systemPrompt = appendProjectInstructions(systemPrompt, d.projectInstructions)` after the fallback assignment and before the `{{.WorkingDir}}` substitution.

### Issue 3: Custom `contains` / `searchSubstring` in test (Code Quality)

**Severity**: Low  
**File**: `vv/configs/project_instructions_test.go`, lines 81-93

The test defines custom `contains` and `searchSubstring` helper functions that exactly duplicate `strings.Contains`. The `strings` package is already imported in `agents/project_instructions_test.go` for the same purpose. This adds unnecessary code.

**Fix**: Replace with `strings.Contains`.

## Observations (No Action Required)

### Observation 1: Duplicated `appendProjectInstructions` logic

The same function body exists in two places: `agents.AppendProjectInstructions` (exported) and `dispatches.appendProjectInstructions` (unexported). The comment in the dispatches version correctly notes this is intentional to avoid a circular import. This is acceptable but worth noting -- if the format string ever changes, both copies must be updated in lockstep.

### Observation 2: Explorer `MaxIterations` capped twice

In `setup.go` line 122, the explorer factory options pass `min(cfg.Agents.MaxIterations, 15)`, and the explorer factory itself also applies `min(opts.MaxIterations, 15)` (explorer.go line 49). This double-capping is harmless but redundant. Not introduced by this change, so no action needed.
