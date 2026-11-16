---
name: goose-worker-recipes
title: Worker recipes — parallel-for, fan-out/fan-in, contained supervision
category: concurrency
tags: [goose, parallel for, fan out, fan in, worker pool, mandelbrot, supervision]
triggers: [goose parallel, parallel for, worker pool, fan out fan in, split work threads]
description: >
  Concrete worker patterns: parallel-for over a big array by chunking,
  fan-out/fan-in with results reassembled by id, per-worker quit sentinels,
  contained workers as supervised restartable tasks, and the snapshot recipe
  for giving workers their own copy of pool data.
source: https://github.com/aardappel/goose/blob/main/docs/tutorial.md#14-threads-that-share-nothing
---

# Worker recipes

## Parallel-for over a big array

Split into chunks; send each chunk (or its index range plus the data it
needs) as a job; merge results in main. The Mandelbrot sample sends one row
per job, and the row's pixels travel *in the message*:

```goose
struct RowJob  { y: i32, width: i32, height: i32 }
struct Row     { y: i32, cells: u8[] }          // flat: the bytes travel along

thread_fn render() {
    loop {
        let j = qget<RowJob>();
        guard j.y >= 0 else { break; }
        var cells: u8[] = compute_row(j);        // built at its destination
        qput(Row { y: j.y, cells: cells });
    }
}
```

Because a flat value can include variable-size parts, the data itself
crosses — no shared output buffer, no locks.

## Fan-out / fan-in

```goose
let n = max(1, hardware_threads());
for i in jobs.len { qput(Job { i: i as i32 }); }
for i in n { qput(Job { i: -1 }); }             // one quit job per worker
var ids: i64[>..] = [];
for i in n { ids.push(thread_spawn(worker)); }
var results: Result[>..] = [];
for i in jobs.len { results.push(qget<Result>()); }
for id in ids { thread_wait(id); }
// reassemble by an id field, or sort(results) { a, b => a.i < b.i };
```

Spawn workers **after** queueing the work (or before — queues buffer
independently of who is listening).

## Supervised restartable tasks (contained workers)

A failed check or `abort(msg)` in a `thread_spawn_contained` worker ends
only that worker; the message goes to the waiter:

```goose
thread_fn serve(attempt: i64) {
    assert(attempt >= 2);                        // a bug the first two hit
    print("attempt ", attempt, " served");
}

fn main() {
    var attempt = 0;
    var why: u8[>..] = [];
    while !thread_wait(thread_spawn_contained(serve, attempt), why) {
        print("attempt ", attempt, " failed: ", why);   // first wait appends the message
        why.clear();                                    // later waits get none: keep it once
        attempt++;
    }
}
```

Stack overflow, `exit(code)`, and aborts on main's thread still end the
program. A worker spawned by a contained worker via `thread_spawn` is not
contained.

## Sharing globals: it does not work, and that is the point

A worker's globals are **its own copies**, taken at spawn. Shared counters
become messages:

```goose
// worker: my_counter += 1   →   main never sees it
// instead: qput(Done { items: n });  main sums the messages
```

## Giving a worker its own pool (snapshot recipe)

Root structs cannot cross to workers. For flat elements, copy the pool's
`items` and allocator metadata explicitly into a worker-local owner (sample
`32_pool_worker_snapshots` shows the recipe) — or keep the deprecated
`reusable` builtins, whose owner transfer to workers still works. Worker-
local library pools (`slot_pool` constructed inside the worker) are fully
supported.

## Sizing

`hardware_threads()` returns a sensible worker count, capped by what the
data stacks have room for (see `--stacks` and `GS_STACK_BUDGET`). Spawning
one worker per hardware thread with one quit job each is the standard
shape.

## Compilation reminder

Threaded programs must be **built** (`goose build`) — TinyCC cannot place
thread-local storage, so `goose run` refuses them. Physics-style libraries
with a main-thread rule are a compile error when reachable from a
`thread_fn`.

## Related skills

- [typed-queues](typed-queues/skill.md)
- [thread-model](thread-model/skill.md)
- [slot-slice-pools](../data-modeling/slot-slice-pools/skill.md)
