---
name: goose-thread-model
title: Threads — thread_fn workers that share nothing
category: concurrency
tags: [goose, threads, thread_fn, no shared memory, workers, separate program]
triggers: [goose threads, thread_fn, worker thread, shared memory, no locks, parallel]
description: >
  Goose threads share nothing: a thread_fn is compiled as a SEPARATE program
  with its own data stacks and its own copies of the globals it uses. No
  locks, atomics or memory orderings exist. Values cross via typed queues of
  flat values (a memcpy). Threads need a real build (TinyCC cannot place
  thread-local storage).
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#22-threads
---

# Threads: workers that share nothing

There is **no shared mutable memory**. A worker is declared `thread_fn`, and
the compiler compiles it and everything it calls as a **separate program**
with its own data stacks and its own copies of the globals it uses — taken
from the spawning program at the moment of the spawn. A worker that
increments a global increments *its own copy*; main's copy is unchanged. No
data races; the language has no locks, atomics or memory orderings.

## The shape

```goose
struct Job    { n: i32 }
struct Result { n: i32, digits: i32 }

thread_fn worker() {
    loop {
        let job = qget<Job>();           // blocks on the queue for Job
        guard job.n >= 0 else { break; } // -1 means stop
        qput(Result { n: job.n, digits: count_digits(job.n) });
    }
}

fn main() {
    let n = max(1, hardware_threads());
    for i in 8 { qput(Job { n: (1 << (i * 3)) as i32 }); }
    for i in n { qput(Job { n: -1 }); }              // one quit job per worker
    var ids: i64[>..] = [];
    for i in n { ids.push(thread_spawn(worker)); }
    var total = 0;
    for i in 8 { let r = qget<Result>(); total += r.digits; }
    for id in ids { thread_wait(id); }
}
```

## The builtins

| Builtin | Meaning |
|---|---|
| `thread_spawn(worker, args…) -> i64` | starts a worker; id, never reused |
| `thread_spawn_contained(worker, args…) -> i64` | worker whose aborts end only that worker |
| `thread_wait(id) -> bool` | blocks until ended and storage released; `false` if contained and aborted |
| `thread_wait(id, msg) -> bool` | first wait appends a contained worker's abort message to `msg` |
| `hardware_threads() -> i64` | sensible worker count, capped by the data-stack budget |
| `qput(v) -> bool` | puts on the queue for v's type; `false` at once if full |
| `qget<T>() -> T` | takes next T, blocking |
| `qget<T>(ms) -> T, bool` | at most ms milliseconds, else zero value and false |
| `qpoll<T>() -> T, bool` | takes one if there is one |
| `qlimit<T>(n)` | queue for T holds at most n messages |

## Flat values only

Whatever crosses (including a worker's arguments) must be **flat**: no
references, slices or relative references at any depth, and not a root
struct. Copying is cheap because a flat Goose value is contiguous — a job
can carry real data, `struct Row { y: i32, cells: u8[] }`, and the crossing
is one memcpy. A `thread_fn`'s parameters have no defaults.

## Rules and sharp edges

- A global that is not flat is an error in any function a `thread_fn`
  reaches — unless it is a `let` whose references all lead to static data
  (as a string literal's do).
- Queued messages outlive their sender. A worker's return releases
  everything it owned whether or not anyone waits. Workers still running
  when `main` returns are killed.
- **Threads need a real C compiler build** (`goose build` / `-o`): the
  in-process TinyCC backend cannot place thread-local storage. `goose run`
  refuses threaded programs.
- Non-standard FFI/C code is outside worker isolation: it can access shared
  native state even though Goose's managed data never shares memory. By
  convention, treat C as thread-safe unless a package says otherwise;
  packages must document non-thread-safe code. Libraries with a
  main-thread rule (`gfx`, `ui`, `audio`, `physics`) are a **compile error**
  when reachable from a `thread_fn`.

## Contained workers

A failed check or `abort(msg)` in a `thread_spawn_contained` worker ends
only that worker and releases everything a return would. The message goes
to the waiter: every `thread_wait` returns `false`, and the first
`thread_wait(id, msg)` appends the message (with source location) to `msg` —
ask for it once and keep it. A supervisor can log and respawn:

```goose
var why: u8[>..] = [];
while !thread_wait(thread_spawn_contained(serve, attempt), why) {
    print("attempt ", attempt, " failed: ", why);
    why.clear();
    attempt++;
}
```

Stack overflow, `exit(code)`, and aborts on main's thread still end the
program. A worker spawned by a contained worker with `thread_spawn` is not
contained. Files opened through `extern fn` stay open.

## Related skills

- [typed-queues](typed-queues/skill.md)
- [worker-recipes](worker-recipes/skill.md)
- [size-classes-and-placement](../memory-model/size-classes-and-placement/skill.md)
