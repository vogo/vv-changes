# Code Review: MCP Credential Filter (P0-4)

Scope: `vage/security/credscrub/`, `vage/mcp/client/`, `vage/mcp/server/`,
`vage/schema/event.go`, `vv/configs/`.

## Findings

### P0 — correctness / security
None that rise to this level. The scanner is well-scoped, `Hit.Masked` caps
at `prefix[:4] + "****"` (and `****` for <8 chars), the block error messages
only emit sorted credential `Type` names (via `SummarizeTypes`), events carry
`Masked` not plaintext, and `ScanJSON` re-marshals the decoded redacted tree
(no text-level replacement attempt that could leak on parse failure after
partial redaction). The fallback-to-text path is only taken on unparseable
JSON, which returns `[]byte(RedactText(...))`; no plaintext-after-redaction
branch exists. Four suspected issues probed and cleared:

- `aws_secret_key` `\b[0-9a-zA-Z/+]{40}\b` — Go's RE2 treats `/` and `+` as
  non-word, so the boundary still fires at `key=wJal...EXAMPLEKEY` (confirmed
  by a scratch program; existing positive test covers this). A string
  bounded by `/.../` won't match — acceptable v1 limitation.
- PEM rule correctly matches PKCS#8 `-----BEGIN PRIVATE KEY-----` (covered
  by `TestDefaultRules_PositiveMatches/pem_pkcs8`).
- JWT rule requires `eyJ` on both segment 1 and segment 2 (verified against
  the regex — `eyJ[...]\.eyJ[...]\.` form). A 2-segment `eyJ.eyJ` input with
  no trailing third segment is rejected (existing `jwt_two_segments_only`
  test).
- Walking JSON — `walkJSON` and `walkMap` both recurse into maps nested
  under slices (via `walkSlice` → `walkMap`) and vice versa. A field-rule
  hit on a key whose value is a non-string falls through to normal recursion
  rather than `continue`, so nested objects under sensitive keys are still
  walked.
- Recursion depth — `encoding/json` enforces a 10 000-depth cap on
  `json.Unmarshal` (confirmed: a 100 000-deep payload returns
  `"invalid character '{' exceeded max depth"`), so `walkJSON` cannot be
  triggered past that bound; no explicit stack guard is needed in this
  layer.

### P1 — applied
1. **`bearer_token` regex lacked a leading `\b`** (`patterns.go`).
   `(?i)bearer\s+[A-Za-z0-9._~+/=-]{16,}` happily matches inside words like
   `cyberbearer <16 chars>`. Fixed to `(?i)\bbearer\s+...`. Verified it still
   catches `Authorization: Bearer ...` and standalone `bearer ...`, and
   rejects `cyberbearer ...` and `Abearer ...`.
2. **`Scanner.Action()` panics on nil receiver** — every other public method
   has a nil guard (the caller uses `c.scanner != nil` but a future
   `Scanner.Action()` call on a stored nil is a sharp edge). Added a nil
   check that returns `ActionLog`.

### P1 — deferred (with rationale)
1. **Inbound loop short-circuit on first block** — `client.CallTool` iterates
   `parts` and returns the first `ActionBlock` immediately, discarding any
   redactions already applied to earlier parts in the same loop. This is
   semantically correct: a block response replaces the entire ToolResult;
   keeping partial redactions would be wasted work. Leaving as-is matches
   the design.
2. **`ScanJSON` fallback-to-text on invalid JSON** — if JSON is unparseable
   AND a text rule hits, the returned `redacted` bytes are themselves
   invalid JSON. Client.CallTool then re-Unmarshals `effectiveArgs` and
   returns `"invalid tool arguments JSON"`. But the original call would
   have produced the same error without the scanner, so there is no
   regression; the call never reaches the wire. Documented limitation.
3. **Duplicate `ScanEvent` in client vs server packages** — the two structs
   have genuinely different fields (client carries `ServerURI`, server does
   not) and live in different conceptual layers. Unifying would require a
   shared type in `credscrub`, coupling the event schema to a security
   package. Keep them split.
4. **`ScanJSONMap` on nil map returns `Action: ActionLog`** instead of
   `s.action`. Cosmetic — callers check `len(Hits) == 0` before dispatching
   on action, so this is never observed.
5. **`Hit.Start` / `Hit.End` are byte offsets**, not plaintext, but could be
   used together with `len(credential)` to hint at position. Acceptable:
   offsets are needed for `RedactText` and are visible only to the same
   process that scanned the text.

### P2 — style / nits (not applied)
- `vage/mcp/{client,server}` both declare `typesSummary(hits)` as private
  helpers; same one-liner. Fine — cross-package dedup would pull event
  formatting into `credscrub`, which is out of scope.
- `ScanEvent.Truncated` is surfaced to callers but no test asserts it
  propagates; integration coverage will catch this.

## Actions Applied

- `vage/security/credscrub/patterns.go` — bearer_token regex now anchored
  with `\b` prefix.
- `vage/security/credscrub/credscrub.go` — `(*Scanner).Action()` now
  nil-safe; returns `ActionLog` when the receiver is nil.

## Test Verification

- `cd vage && go test ./security/... ./mcp/... ./schema/...` — PASS.
- `cd vv && go test ./configs/...` — PASS.
- `cd vage && make lint` — 0 issues.
- `cd vv && make lint` — 0 issues.

## Test Coverage Gaps (flagged, not authored here)

- No unit test asserting that when scanner BOTH redacts args AND detects
  credentials in the returned text of the same `CallTool`, both firings
  emit independent events and both redactions apply. Belongs in
  `integrations/mcp_tests/` per design §6.4.
- No test asserting `ScanEvent.Truncated` propagates from `ScanResult` to
  callback for oversized inputs.
- No test for env-var parse failure (`VV_MCP_CREDFILTER_ENABLED=notabool`)
  — the code logs and ignores; a `Load` test asserting the config remains
  default would be cheap.
