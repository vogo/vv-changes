# Complexity Assessment

| Phase | Decision | Rationale |
|---|---|---|
| improver | **INCLUDE** | New `vage/vector/` package introduces a public interface (VectorStore + Embedder) that future real backends will implement; the interface shape (dimension lock, score semantics, namespace, Embedder shape) deserves a senior second opinion before downstream impls solidify. Also informs P10 SessionTree's `vector_recall(...)` dependency. |
| reviewer | **INCLUDE** | Substantive diff (~600+ LOC new code), new public API surface, concurrency primitives in MapVectorStore, fail-open contract that must not regress Builder behaviour. |
| tester | **SKIP** | All acceptance criteria are unit-testable; the design's §7 plan already covers happy path / fail-open / TopK / MinScore / Filter / Builder integration via colocated `*_test.go`. No external boundary (real vector backend, real embedder API) to integration-test in this iteration; integrations/ tests would duplicate unit coverage. Developer phase runs unit tests + lint inline. |

**Effective pipeline**: analyst → designer → improver(†) → developer → reviewer(†) → documenter
