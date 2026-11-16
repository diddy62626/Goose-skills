---
name: goose-recursion
title: Recursion — recursive fn, the growable-data rule, shared scratch
category: functions
tags: [goose, recursion, recursive fn, tree walk, scratch stack, parser]
triggers: [goose recursion, recursive fn, recursive tree, growable data recursion, recursive descent]
description: >
  Recursion is opt-in: mark the cycle's entry recursive fn, write out every
  signature in the cycle, and no function in the cycle may hold growable
  data across a call back into the cycle. Three workarounds: pass the
  accumulator in, scope the scratch to end before the call, share one
  grow-shrink stack truncated to a mark. Plus the detailed rules.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#19-recursion
---

# Recursion

Recursion is **opt-in and annotated**. The entry function of every recursive
cycle is marked `recursive fn`, every function in the cycle has a fully
explicit signature, and the key restriction is:

> No function in the cycle may call back into the cycle while one of its
> non-fixed locals is in scope.

A growable or variable-size local holds a data stack from its declaration to
the end of its block; if one were in scope at a recursive call, every
activation would need a stack of its own. Recursion depth then uses only
native stack, and exhausting it is a clean abort.

## The three ways around the restriction

**1. Pass the accumulator in** — own it outside the cycle:

```goose
recursive fn in_order(n: Node&, out: i32[>..]&) {
    let l = n.left;
    if l { in_order(l, out); }
    out.push(n.key);
    let r = n.right;
    if r { in_order(r, out); }
}
```

**2. Scope the scratch so it ends before the call:**

```goose
recursive fn show(n: Node&, depth: i64) {
    {
        var line: u8[>..] = [];              // freed at this block's end
        for i in depth { line.append("  "); }
        format(line, n.key);
        print(line);
    }
    let l = n.left;
    if l { show(l, depth + 1); }
    let r = n.right;
    if r { show(r, depth + 1); }
}
```

**3. Share one grow-shrink stack across the recursion**, truncating to a
mark before returning (a *balanced* call):

```goose
recursive fn walk(scratch: i64[>..<]&, depth: i64) -> i64 {
    let mark = scratch.len;
    scratch.push(depth);
    let mine = scratch[mark..];
    var total = 0;
    if depth < 3 { total = walk(scratch, depth + 1); }
    total += mine[0];
    scratch.resize(mark);
    return total;
}
```

## What recursive-descent parsers do instead

The state becomes locals of the **non-recursive entry function** — or a
struct of references to those locals (`Ctx`) — and the grammar rules are
nested functions or take the context by reference. This is how the samples'
parsers are structured, and it is nicer than it sounds: fresh state per
call, freed all at once at the driver's scope exit.

## The rules in detail

- **Signatures.** No result type is inferred across a cycle; none may be
  inferred as a reference.
- **Parameters.** No function in a cycle has a by-value non-fixed parameter:
  it takes a reference or a slice.
- **Stores.** A reference rooted at a cycle function's own local may be
  passed down but not stored anywhere or returned (activations of one local
  cannot be told apart). References rooted outside the cycle are
  unrestricted — a pool passed in by reference, a local of an enclosing
  non-recursive function, a global. Every recursive call must pass such a
  pool on as the entry call did.
- **Results.** A cycle function may return the result of a recursive call;
  roots are worked out in rounds over the cycle.
- **Polymorphic recursion.** A recursive call may instantiate its callee
  with other type arguments, but the instantiations must be finite — a call
  whose types grow each round is rejected once it would put more than 16
  specializations of one function in progress.
- **Temporaries.** At a call into the cycle, no non-fixed temporary may be
  live: a `str(…)` argument, or the call inside a `print`, is an error.

## Related skills

- [nested-functions](../nested-functions/skill.md)
- [grow-shrink-stacks-queues](../../arrays/grow-shrink-stacks-queues/skill.md)
- [pattern-catalog](../../idioms/pattern-catalog/skill.md)
