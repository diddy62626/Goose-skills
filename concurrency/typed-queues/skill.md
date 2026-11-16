---
name: goose-typed-queues
title: Typed queues — qput/qget/qpoll/qlimit patterns
category: concurrency
tags: [goose, queues, qput, qget, qpoll, qlimit, message passing, backpressure]
triggers: [goose queue, qget, qput, qlimit, message queue, worker communication]
description: >
  One typed queue per value type, selected by the type. Blocking qget,
  polling qpoll, timeout qget<T>(ms), backpressure qlimit + qput returning
  false. Design patterns: quit sentinels, request/response with id fields,
  bounding queue memory.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#22-threads
---

# Typed queues

Values cross between worker programs through **typed queues** — one queue
per type, selected by the type. Whatever crosses must be flat (no
references/slices at any depth), which makes the crossing a single memcpy of
contiguous bytes.

## The API

```goose
qput(v) -> bool            // enqueue; false at once if the queue is full
qget<T>() -> T             // blocking take
qget<T>(ms) -> T, bool     // at most ms milliseconds; else zero value and false
qpoll<T>() -> T, bool      // take one if available, else zero value and false
qlimit<T>(n)               // cap the queue for T at n messages
```

`qput(v);` as a statement ignores the result. There is no per-queue object
to pass around: the type *is* the address.

## Patterns

**Quit sentinel per worker** — put one stop job per worker before or after
spawning:

```goose
for i in n { qput(Job { n: -1 }); }
```

**Request/response** — a result carrying the job's id reassembles ordering:

```goose
struct Render { frame: i32, y: i32, cells: u8[] }   // flat: carries its data
thread_fn renderer() {
    loop {
        let j = qget<RenderJob>();
        guard j.frame >= 0 else { break; }
        qput(Render { frame: j.frame, y: j.y, cells: render_row(j) });
    }
}
var rows: Render[>..] = [];                 // main collects; sort by y/frame
```

**Backpressure** — turn work away instead of queueing without end:

```goose
qlimit<Job>(256);
if !qput(Job { ... }) { /* queue full: drop, retry, or report */ }
```

Bound the memory a queue may hold by setting its limit to memory ÷ size of
a typical message.

**Timeouts and polling** — a supervisor that also does work, or a graceful
shutdown check:

```goose
let job, got = qpoll<Job>();
if got { handle(job); } else { idle_work(); }
```

## Semantics to remember

- A queue has no message limit until `qlimit` gives it one.
- Queued messages outlive their sender.
- A worker's return releases everything it owned, whether or not anyone
  waits; workers still running when `main` returns are killed.
- `qget<T>()` on an empty queue **blocks**; use `qpoll`/`qget<T>(ms)` when
  main must keep working.
- Queue types must be flat: `struct Row { y: i32, cells: u8[] }` crosses as
  one copy including its bytes.

## Related skills

- [thread-model](thread-model/skill.md)
- [worker-recipes](worker-recipes/skill.md)
