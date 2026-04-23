# Design Review — P1-5 结构化对话轨迹落盘（JSONL）

Reviewer: improver phase. Scope check: P1-5 JSONL export via AsyncHook only. No
OTEL / Langfuse / redaction / SQLite / resume scope creep.

## Overall assessment

The design is tight, well-scoped, and leans on existing vage infrastructure
(`hook.Manager`, `schema.Event`, `taskagent.WithHookManager`) exactly as the
requirement intends. It correctly resists the temptation to build a new
observability layer. The bulk of this review is *keeping* that posture and
trimming a few small items that do not pay for themselves at MVP.

Verdict: accept the design with the small deltas below. No structural rework
required.

## Accepted improvements (applied in new `design.md`)

### A1. Drop `part` pre-open on `ensureFile`; open-on-rotate only
Rationale: the design already opens the file lazily on first event. The
`rotate(sid)` path described does `Sync+Close` old then on next write
`ensureFile` opens `<sid>.<part>.jsonl`. Good. But the wording implied a
separate "rotate then re-ensure" sequence. Clarify it's a single atomic step
inside the write branch: detect `written+len(line) > max` → `rollSession(sid)`
(close old, bump part, open new, zero counter) → then `Write(line)`. One
function, one lock region. Avoids the "double ensureFile" read in prose.

### A2. Make `MaxFileBytes == 0` explicit: "disabled / no rotation"
Already noted in struct comment; elevate to a one-line rule in the rotation
section so the developer doesn't need to infer. Negative values are coerced to
default (not zero) — keep defaults deterministic.

### A3. Clarify shutdown channel-close semantics
The current text says "close(ch) after which consumer drains". That is correct
but under-specified vs. the senders: `hook.Manager.Dispatch` may race with
`close(ch)` and panic on send. Document that **`Stop` must first deregister /
signal the Manager to stop dispatching to this hook, then close `ch`**. In
practice vage's `hook.Manager.Stop` already sequences this (stop dispatch →
drain hooks). Design should state the ordering contract explicitly so the
developer wires `Stop` via Manager, not directly on the hook, and does NOT call
`close(h.ch)` from `JSONLHook.Stop` if Manager already owns the channel
lifecycle. Pick one owner. Recommended: **Manager owns the channel** — the
hook's `Stop` only flushes/closes files; the consumer goroutine exits when the
channel Manager closes. This is the least surprising model and matches how
`AsyncHook` is typically implemented.

If inspection of `vage/hook` shows the hook owns the channel, flip to that
model and state it. The design should not be ambiguous about ownership.

### A4. Remove `RedactFunc` field from MVP
The design text says "预留 `RedactFunc` 字段但 MVP 不暴露配置". That still adds
surface area (a field, a nil check, a test matrix item) for zero MVP benefit.
Redaction is P3-5. **Do not add the field at all.** When P3-5 lands it will
design the redaction seam properly (likely a pipeline, not a single func). YAGNI.

### A5. Defer file path explicitness in `InitResult`
Design adds `InitResult.HookManager`. That is necessary. Do **not** also
surface `InitResult.TraceDir` or file paths — they're internal to the hook. If
a future feature needs them, add then. Keep the `InitResult` delta minimal:
one field.

### A6. Lifecycle: start hook inside `setup.Init`, not in `main.go`
Currently the design has `main.go` calling `HookManager.Start` after
`setup.Init` returns. Simpler: call `Start` inside `setup.Init` as the final
step (after agents are built but before return), and return a `Shutdown
func(ctx)` closure alongside `HookManager` for `main.go` to `defer`. This:
  1. Keeps all trace-assembly in one place (cohesion).
  2. Lets `setup.Init` fail atomically if hook init fails (rollback already-
     built state is easier than in main.go).
  3. Each of the three modes (CLI/HTTP/MCP) just calls
     `defer initResult.Shutdown(ctx)` without knowing about HookManager.

This is a small refactor but yields cleaner mode wiring than "main.go branches
on `HookManager != nil`" in three places. If the existing `SetupResult`
already has a `Shutdown`/`Close` convention, compose into it; otherwise add one.

### A7. Test #6 is the acceptance-critical one — call it out
The "disabled path" test (no Manager, no goroutine, no open file) is the
guardrail for US-2 (zero cost when off). Promote it from list item 6 to a
first-class "MUST pass" test with a specific assertion: after `setup.Init`
with `trace.enabled=false`, `runtime.NumGoroutine()` delta is 0 AND no file
handle under `traces/` (verify by listing dir). Without a concrete assertion
"no extra goroutine" is vibes.

### A8. Integration test path — reuse existing pattern, don't invent new dir
Design proposes `integrations/traces_tests/tracelog_tests/`. Fine, but align
with the **existing** `integrations/traces_tests/budget_tests/` sibling layout
the design already references. One sentence in the design confirming the
parallel is enough — avoids developer guessing.

## Rejected / not-applied suggestions (considered, not worth it)

### R1. "Add a schema version field to each line"
Tempting (forward-compat). Rejected: `schema.Event.Type` already discriminates
and the sealed `EventData` interface gives us typed evolution. Adding a
`"v":1` field to every line is churn without a concrete reader that branches
on it. If P2-14 resume or P3-5 export later needs versioning, add it there
and let older files default to v0.

### R2. "Batch writes / bufio.Writer for throughput"
Rejected for MVP. The requirement explicitly calls this out as "not doing"
(line 265 of raw design). `os.File.Write` + periodic `Sync` is fine for the
event volume of an interactive agent. Batching adds `flush-on-idle` timer
complexity and makes crash-safety reasoning harder. If profile shows pain,
wrap with `bufio.NewWriterSize` later — trivial retrofit.

### R3. "Write sidecar metadata file per session (model, start_time, cwd)"
Rejected: `agent_start` / first `llm_call_end` already carries this. Sidecar
file adds a second I/O path and a second consistency problem for zero win.
If P2-14 resume needs an index, build it there.

### R4. "Guard against path traversal via project hash, not just session id"
The design already hashes cwd to a fixed-length base32 token. Traversal is
structurally impossible. Reject the paranoia double-check.

### R5. "Expose `event_types` allowlist in MVP"
The raw requirement mentions `trace.event_types` as "MVP 可选". Reject for MVP.
Full-firehose is what P3-5 wants anyway; filtering is a consumer concern
(`jq`). Shipping a filter means designing the filter DSL, testing combinations,
and documenting it. Zero demand today.

### R6. "Rotate by date (daily file) instead of by size"
Rejected. Session-scoped files are the correct primitive for P2-14 resume and
P3-5 export. Daily rotation cuts across sessions and forces downstream to
re-stitch. Size-based part rotation is the right MVP choice.

### R7. "Add Prometheus/metrics counters for drops and writes"
Rejected. `slog.Warn` on drop is enough visibility for MVP. Metrics wiring is
a separate story and vv has no metrics backend configured anyway.

## Minor nits (applied inline, not worth bullets)

- Use `time.RFC3339Nano` formatting for timestamps if `schema.Event` doesn't
  already mandate it — requirement says "毫秒精度"; whatever `schema.Event`
  marshals to is the contract, don't re-format in the hook.
- `sanitizeSessionID` 128-char cap is fine but document *why* 128 (filesystem
  name limits vary; 255 is common floor, 128 is conservative).
- The `projecthash.go` 12-byte / base32 choice is good; one sentence saying
  "collision probability negligible for the ~10^3 projects a user might
  traverse" would put the decision to bed for the reviewer.
- `setup.go` L516 stale comment — design already notes updating it. Keep.

## Scope guard (explicitly NOT in this ticket)

Confirmed the following remain out-of-scope and the refined design does not
introduce them:

- OTEL / Langfuse export (P3-9)
- PII / redaction scrubbing (P3-5)
- SQLite store (P1-6)
- Session resume / replay (P2-14, P4-12)
- Cross-process session merge (P1-6)
- Reusing `vv/debugs/` (different trigger, different format, correctly separate)

## Summary of deltas applied to `design.md`

1. Channel-ownership contract stated explicitly (A3) — Manager owns the
   channel; hook's `Stop` only flushes files.
2. Rotation described as a single `rollSession(sid)` step (A1) + explicit
   `MaxFileBytes==0` rule (A2).
3. `RedactFunc` removed from MVP surface (A4).
4. `InitResult` delta trimmed to just `HookManager` + a `Shutdown` closure
   (A5, A6); `main.go` wiring simplified to `defer initResult.Shutdown(ctx)`.
5. Disabled-path test promoted with concrete goroutine-count / file-handle
   assertions (A7).
6. Integration test location aligned with `integrations/traces_tests/budget_tests/`
   sibling convention (A8).

No structural changes to module layout, data format, hashing, or dispatch path.
