# Code Review: Conversation Context Compression

## Summary

Overall the implementation faithfully follows the design document. The three-layer defense (tool truncation, proactive compact, reactive emergency compact) is well-structured with clear separation between the framework layer (`vage/`) and application layer (`vv/`). Test coverage is solid across all new components.

Three issues were found and fixed; two additional observations are noted for future consideration.

## Issues Found and Fixed

### 1. Data Race in `invokeAgent` (Critical)

**File:** `vv/cli/cli.go`

The `invokeAgent` method runs as an async `tea.Cmd` (goroutine) but directly mutates `m.app.history` and `m.app.estimatedTokens`, which are also read/written by the single-threaded Bubbletea `Update` loop. This is a data race.

**Fix:** Snapshot `history`, `estimatedTokens`, and `sessionID` before entering the goroutine. The compaction result is sent back via `compactionDoneMsg` (extended with `compressed` and `newTokens` fields) so that the `Update` handler applies the state change safely on the main goroutine.

### 2. Data Race in `manualCompactCmd` (Critical)

**File:** `vv/cli/memory.go`

Same pattern: `manualCompactCmd` runs as an async `tea.Cmd` and directly mutates `m.app.history` and `m.app.estimatedTokens`.

**Fix:** Introduced a new `manualCompactResultMsg` type. The goroutine now returns the result via this message, and the `Update` handler applies state changes. Also simplified the logic: the previous code called `CompactIfNeeded` with `threshold=0` (which always triggers compaction since any token count > 0), then had a dead-code fallback to force compact. Replaced with a direct `Compact` call.

### 3. UTF-8 Truncation Safety in `TruncatingToolRegistry` (Minor)

**File:** `vage/tool/truncate.go`

`part.Text[:maxChars]` can split a multi-byte UTF-8 rune, producing invalid UTF-8 in the truncated output. Tool results may contain non-ASCII content (CJK text, Unicode symbols, file contents).

**Fix:** Added a backward scan to find the nearest valid rune boundary using `utf8.RuneStart`.

## Observations (Not Fixed)

### 4. HTTP Mode Compactor Not Used

**File:** `vv/httpapis/http.go`

The `compactor` parameter is accepted by `Serve()` but never used within the function body. The design (Section 2.8) calls for proactive compression before dispatching in HTTP mode, plus reactive compression on overflow. This is currently unimplemented.

This is noted but not fixed because the HTTP handler currently delegates to `service.New()` from the vage framework, and injecting compression would require changes to the request pipeline that go beyond a simple fix. The compactor parameter serves as a placeholder for a future implementation.

### 5. `renderSummaryMessage` Never Called in Production

**File:** `vv/cli/render.go`

The function is defined and tested, but no production code path calls it. The design (Section 2.9) says messages with `Metadata["compressed"] == true` should be detected during rendering and displayed with this function. Since compressed history messages are not re-rendered to the user (they exist only in the internal `history` slice sent to the LLM), this is currently harmless. If a future feature displays conversation history, this detection logic will need to be wired up.

## Code Quality Notes

- **Token estimation**: The `len(text)/4` heuristic is adequate for budget checks but will consistently undercount for languages with multi-byte characters (e.g., CJK where 3 bytes per character yields ~0.75 tokens per character, not 0.25). The 10% safety margin partially compensates.
- **Test coverage**: All new components have thorough unit tests. The compactor tests cover edge cases (empty messages, canceled context, nil compactor, existing summaries, custom estimators). Overflow detection tests cover all provider patterns.
- **Thread safety claims**: `ConversationCompactor` is documented as "safe for concurrent use" which is accurate -- it holds no mutable state after construction. The `With*` builder methods mutate during construction only.
- **Config design**: Using `*float64` for `CompressionThreshold` to distinguish "not set" from zero is a good pattern, with `EffectiveCompressionThreshold()` providing a clean API.

## Files Modified by Review

| File | Change |
|------|--------|
| `vv/cli/cli.go` | Fixed data race in `invokeAgent`; snapshot shared state before goroutine; extended `compactionDoneMsg` handler to apply state |
| `vv/cli/memory.go` | Fixed data race in `manualCompactCmd`; return result via message; simplified dead-code logic |
| `vv/cli/messages.go` | Added `manualCompactResultMsg` type; extended `compactionDoneMsg` with compressed/newTokens fields |
| `vage/tool/truncate.go` | Added UTF-8 rune boundary check before truncation |
