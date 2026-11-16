# concurrency

**Working with data** — Threads that share nothing: thread_fn, typed queues, worker recipes.

3 skills. Load any of them as `concurrency/<skill>/skill.md`.

- [`goose-thread-model`](thread-model/skill.md) — Threads — thread_fn workers that share nothing: > Goose threads share nothing: a thread_fn is compiled as a SEPARATE program with its own data stacks and its own copies of the globals it uses. No locks, atomics or memory orderings exist. Values cross via typed queues of flat values (a memcpy). Threads need a real build (TinyCC cannot place thread-local storage).
- [`goose-typed-queues`](typed-queues/skill.md) — Typed queues — qput/qget/qpoll/qlimit patterns: > One typed queue per value type, selected by the type. Blocking qget, polling qpoll, timeout qget<T>(ms), backpressure qlimit + qput returning false. Design patterns: quit sentinels, request/response with id fields, bounding queue memory.
- [`goose-worker-recipes`](worker-recipes/skill.md) — Worker recipes — parallel-for, fan-out/fan-in, contained supervision: > Concrete worker patterns: parallel-for over a big array by chunking, fan-out/fan-in with results reassembled by id, per-worker quit sentinels, contained workers as supervised restartable tasks, and the snapshot recipe for giving workers their own copy of pool data.
