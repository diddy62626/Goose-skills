---
name: goose-lifetimes-roots
title: Lifetimes — roots, stores, and reading the errors
category: references
tags: [goose, lifetime, borrow checker, root, stores, holders, errors]
triggers: [goose lifetime, borrow checker, root variable, reference outlives, lifetime error]
description: >
  Goose's entire "borrow checker" is one rule: a reference must not outlive
  the variable that owns its target, and must never observe its target at a
  wrong type. No annotations, no exclusivity. Roots are inferred and
  functions are specialized per root. The store/return/branches/rebinding/
  temporary/holder rules, and how to read the two-ended error messages.
source: https://github.com/aardappel/goose/blob/main/docs/reference.md#12-lifetimes-roots-stores-and-writability
---

# Lifetimes: roots, stores and writability

This is the whole of Goose's "borrow checker":

> A reference or slice must not outlive the variable that owns its target,
> and must never observe its target at a wrong type.

There are **no lifetime annotations anywhere** and **no aliasing or
exclusivity rules** — two references to the same thing are fine, because
without shared-memory concurrency aliasing alone cannot break type safety.

## Roots

Every reference and slice has one or more static **roots**: a local or
global variable (or a temporary) that bounds the lifetime of its target. A
root is **exact** when that variable's own storage contains the target,
inexact when it only bounds the lifetime. Roots are inferred, never written;
functions are specialized per distinct root — the precision of Rust
lifetimes with no syntax for them.

## The rules

- **Store.** A reference may be stored into a location only if its root
  outlives the location's owner. A declaration stores its value into the
  variable it declares.
- **Return.** A returned reference's root must be visible to the caller: a
  caller-supplied argument, a global, or the function's own in-place
  constructed result. A part of the function's own local dies with it.
- **Branches.** The value of an `if`/`match`/`block`/loop is received after
  its branches' scopes ended, so it may not be rooted at a variable declared
  inside a branch.
- **Merged values.** A value from several places (branches, rebindings,
  returns) has the roots of all of them and must satisfy each rule for every
  root.
- **Rebinding.** A reference variable commits to its first binding's root;
  `.=` may rebind within the same root or to a root at the same scope depth
  (the common "walk a container" case). Anything else needs a new variable.
- **Temporaries.** A literal, call's result, `copy`, or control construct's
  value lasts for the rest of its statement. A view into one may be passed
  down but not stored or returned: `let s = f()[..];` is an error;
  `let t = f(); let s = t[..];` is fine.
- **Holders.** A holder (struct with reference fields, array of slices) is
  rooted at what was stored into it; the checker records every such store,
  so it also catches a shrink or scope exit that would strand a reference
  inside a holder.

```goose
fn longest(a: u8[:], b: u8[:]) -> u8[:] { if a.len >= b.len { a } else { b } }

var outer: u8[>..] = [];
outer.append("outer");
var w = outer[..];
{
    var inner: u8[>..] = [];
    inner.append("inner text");
    // w = longest(outer, inner);   // error: result rooted at inner, which dies first
}
```

Move the declaration of `w` inside the block, or copy the text out.

## Reading a reference out of a container

A container identifies a scope, not the storage its contents point into. The
checker derives the root again: each variable in scope whose storage could
hold the pointee's type is a **candidate**; one candidate → rooted exactly;
several → rooted at all inexactly. Relative links `in pool` point exactly
into `pool`; a local holder whose stores are all known is rooted at what was
stored. Operations needing exact identity (storing a relative link,
`index_of`, shrink attribution) report candidates when the checker cannot
choose: "`n` was read out of `slots` and may point into `pool` or `spare`".

## Reading the errors

When the checker rejects something, **read the whole message**. It names
both ends: the reference *and* where it was bound, the holder *and* the line
that stored into it, the instantiation *and* the call chain that produced
it. Use those locations to decide which end to change: use the view for the
last time before the shrink, move the shrink, or move the declaration.

## Related skills

- [const-writability](const-writability/skill.md)
- [slices](../arrays/slices/skill.md)
- [reading-errors](../idioms/reading-errors/skill.md)
