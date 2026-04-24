# Auto plan — P1-7 · Parallel tool calling

## Phase decisions

- **improver: SKIP** — the design uses a standard Go pattern (WaitGroup + channel semaphore), surfaces defaults explicitly, and rejects obvious alternatives (errgroup's cascading cancel, per-tool opt-out metadata) with reasons. No material second-opinion lever.
- **reviewer: INCLUDE** — concurrency in core shared code (TaskAgent) with subtle ordering guarantees; a clean extra set of eyes on the fan-out logic and event/message ordering is worth the cost.
- **tester: INCLUDE** — concurrency-dependent acceptance criteria (timing, event order, error isolation) deserve explicit integration coverage beyond unit tests.

## Effective pipeline

`analyst → designer → developer → reviewer → tester → documenter`
