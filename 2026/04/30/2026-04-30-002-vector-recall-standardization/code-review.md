# Code Review — Vector Recall Standardization

Scope: `vage/vector/{vector,mapstore,embedder}.go` (+ tests), `vage/context/sources_vector.go` (+ test), `vage/context/source.go` (constant addition).

Tests pass before and after changes; `make lint` reports 0 issues.

---

## Findings

### [applied] vage/vector/mapstore.go:108–119 — `Add` did not deep-copy `Metadata`

**Issue.** The defensive copy on `Add` cloned `Embedding` but stored `doc.Metadata` as-is. A caller mutating its metadata map after `Add` would silently corrupt the stored document — a real footgun for a foundation-grade store, especially given the package documents itself as concurrent-safe.

**Fix.** Added shallow `cloneMetadata` (the typical metadata payload is scalars; deep-cloning arbitrary `any` is too expensive and out of scope). Mutation of nested mutable values is now a documented, narrow corner case.

### [applied] vage/vector/mapstore.go:188–200 — `List` leaked Embedding/Metadata aliases

**Issue.** `List` ranged over `s.docs` and appended each `Document` by value, but `Embedding` (slice header) and `Metadata` (map header) still pointed at the stored data. Callers mutating the listed copies would corrupt the store under the read lock — a concurrency hazard that the doc-comment ("safe for concurrent use") explicitly disclaims.

The precedent in `vage/memory/mapstore.go::List` returns `StoreEntry{Value: any}` and does not clone, but `memory.MapStore` is documented as **not** safe for concurrent use, so the bar there is lower.

**Fix.** Clone `Embedding` and `Metadata` per returned doc.

### [applied] vage/vector/mapstore.go:128–174 — `Search` shared the same alias hazard

**Issue.** Same root cause as List: returned `SearchHit.Document.Embedding` and `Metadata` aliased the stored maps/slices.

**Fix.** Clone them post-TopK so we only pay for hits the caller actually receives. Predicate still sees the underlying stored values (it runs while holding the read lock — mutating-from-predicate would race with anything that takes the write lock and is therefore documented-not-supported behaviour).

### [applied] vage/context/sources_vector.go — user-supplied `Render` panic could violate fail-open

**Issue.** `VectorRecallSource.Fetch` is documented "errors short-circuit to fail-open and never bubble up." A panicking user-supplied `Render` (e.g. nil-pointer in caller code) would unwind through `fitToBudget` → `Fetch` → `Builder.Build`, taking the whole turn down. The contract said "no error escapes"; it said nothing about panics, but the spirit is "Builder never gets killed by an optional source."

**Fix.** Wrap `render` in a `recoveringRenderer` that swallows panic, logs it, and returns `""`. The empty render then funnels into the documented "empty render → Status=skipped" branch. Internal `defaultHitsRender` is also wrapped uniformly — cheap and keeps the call site simple.

### [applied] Test additions

- `TestMapVectorStore_DefensiveCopy_Metadata` — exercises both directions: caller-mutates-after-Add and caller-mutates-listed-doc.
- `TestMapVectorStore_Search_DefensiveCopy` — same for Search results.
- `TestMapVectorStore_CtxCancellation` — Add/Search/Delete/List all honour `ctx.Err()` (filling a documented gap).
- `TestMapVectorStore_ConcurrentAddSearch` — interleaved writers + readers under `-race`.
- `TestVectorRecallSource_RenderPanicRecovered` — panic in user renderer surfaces as `Status=skipped`.

---

## Items Verified, Not Changed

- **Filter ordering in `Search`** (mapstore.go:147–157): Predicate runs **before** cosine (perf win); MinScore runs **after** score is computed. Order is correct.
- **Cosine NaN handling** (mapstore.go:229–242): zero-vector short-circuit returns 0; for finite f32 inputs no NaN can be produced. Non-finite inputs (Inf/NaN from a buggy embedder) would propagate — this is cooperative-input territory and not the package's job to police.
- **Concurrency**: `Add`/`Delete` write-lock; `Search`/`List`/`Len`/`LockedDimension` read-lock; no nested locks, no recursion. Safe.
- **`HashEmbedder` non-ASCII**: `unicode.IsLetter`/`IsDigit` accept Han ideographs; emoji are split out. No panic risk; CJK without spaces forms one giant token, which is fine for a fixture.
- **`fitToBudget` convergence**: `maxBytes /= 2` integer-division ladder (4→2→1→0) guarantees termination; the `if maxBytes <= 0` exit is reached for pathological tiny budgets.
- **`defaultQuery` multimodal**: `Content.Text()` returns concatenation of text parts only — image-only user messages return `""`, the `TrimSpace` skip works, the walk continues backwards. Verified against `aimodel/schema.go::Content.Text`.
- **Token estimator wrapping**: dropping the `aimodel.Message` wrap in favour of `memory.EstimateTextTokens(text)` would lower coupling, but the user-overridable `TokenEstimator` field has signature `func(schema.Message) int`, so the wrap is unavoidable on the override path. Keeping a single code path is simpler than branching.
- **Source-name constant placement**: `SourceNameVectorRecall` sits with peers in `vage/context/source.go` and follows the existing naming convention.
- **Fail-open contract**: every `return` from `Fetch` returns `nil` error; Status is one of `ok`/`skipped`/`error`/`truncated` with `Note` populated. Matches `WorkspaceSource` / `SessionMemorySource` style.

---

## [suggested] (non-blocking, follow-up worthy)

- **`ErrEmptyQuery` reuse on Add path** (`mapstore.go:101–103`): when `Add` is called with an empty `Embedding`, the error is `ErrEmptyQuery`, which semantically describes a Search query, not an Add. A separate `ErrEmptyEmbedding` (or wrapping with `fmt.Errorf("%w: empty embedding", ErrEmptyQuery)`) would read better in logs. Cosmetic; left alone to avoid breaking anyone who already pattern-matches `ErrEmptyQuery`.
- **`fitToBudget` dead-branch**: the inner `if len(hits) == 0 { return "", hits, true }` (sources_vector.go) is unreachable — the outer `Fetch` already returns when `len(hits) == 0`, and the trim loop stops at length 1. Harmless guard, but could be deleted.
- **Embedder-returning-wrong-dim coverage via `VectorRecallSource`**: there is no test that wires a misbehaving Embedder (returns a 4-dim vector against a 3-dim store) and asserts the Source surfaces `Status=error` with `ErrDimensionMismatch` text. The behaviour falls out of `Store.Search`'s `ErrDimensionMismatch` so it is technically covered by `TestVectorRecallSource_FailOpenOnSearchError`, but a direct e2e would be more defensive.
- **Predicate mutating stored Metadata**: technically possible because Predicate runs while holding the read lock and sees the underlying map. Not a footgun in practice but worth a one-line godoc note ("Predicate must treat Document as read-only").
