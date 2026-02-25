# Stacked Borrows Model

[Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf) is an aliasing model for [Rust](./rust.md) programs.
It is a dynamic model that ensures that memory accesses follow the borrowing semantics that are statically enforced by the borrow checker.
The Stacked Borrows model tracks memory accesses without relying on reasoning about lifetimes.
The main motivation for this model is to validate Rust `unsafe` programs at runtime.
For this purpose, [Miri](https://github.com/rust-lang/miri) utilizes the model for alias validation.
This section focuses only on a subset of this model.

## Overview

Based on the reference overview, Rust’s [borrow checker](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) enforces a couple of rules for mutable and immutable aliases, which achieve the [borrow checker](./rust.md#borrowing)'s first goal that at each point of execution, the program has access to either one mutable reference or multiple immutable references for a value at the same time.
The Stacked Borrows model restates those rules to achieve the same goal without depending on lifetimes.

First, in the Stacked Borrows model, when a mutable reference is borrowed from a referent, which can be a value owner or another mutable reference:
*every use of that reference (and anything derived from it) must occur before the next use of the referent after the reference was created.*
Here, *use* includes moving, dropping, borrowing, re-borrowing, dereferencing, updating (mutating), reading, and similar operations.

In the following example, the program uses `y` after `x`, where `y` is a reference borrowed from `x` as its referent.
This usage results in an error and violates the Stacked Borrows model:

```Rust
let mut local = 0;
let x = & mut local;
let y = & mut *x; // re-borrow `x` to create `y`
*x = 1; // use `x` again
*y = 2; // error: `y` used after `x` got used!
```

Second, based on Stacked Borrows Model, when an immutable reference is borrowed from a referent, which can be a value owner, a mutable reference, or another immutable reference:
*every use of the immutable reference (and everything derived from it) must occur before the next mutating use of the referent (after the reference got created), and moreover the reference must not be used for mutation.*

```Rust
let mut local = 42;
let x = & mut local;
let ir1 = &*x; // immutably re-borrow `x` to create `ir1`
let ir2 = &*x; // immutably re-borrow `x` to create `ir1`
let val = *x; // reading `x`
let val = *ir1; // reading `ir1`, ok
let val = *ir2; // reading `ir2`, ok
*x += 17; // mutate `local` through `x`
let val = *ir1; // error: `ir1` is used after its reference, `x`, is used mutably.
```

In the example above, reading from `x` and then from `ir1` and `ir2` is valid.
However, after writing to the value of `local` through `x`, the program can no longer read from `ir1` or `ir2`.

## Formalization

The subset of the Stacked Borrows model that is in imem's interest models memory as a mapping from locations to pairs consisting of a scalar and a stack.

$$
\text{Mem}: \text{Loc} \rightarrow \text{Scalar} \times \text{Stack}
$$

Each scalar is either a value, an integer, or a pointer. A pointer \( \text{Pointer}(l, t) \) is defined as a pair composed of a location \( l \) and a unique pointer identifier \( t \).  

$$
\text{Scalar}: \text{Pointer}(l, t) \mid z \quad \text{where } z, t \in \mathbb{Z} \\
$$

Each location is associated with a stack, which is an ordered collection of tags. Each tag is either a unique tag, \( \text{Unique}(t) \), or a shared tag, \( \text{Shared}(t) \). Both kinds of tags contain a unique pointer identifier \( t \).

$$
\text{Tag}: \text{Unique}(t) \mid \text{Shared}(t)
$$

$$
\text{Stack}: \text{List}(\text{Tag})
$$

When the program allocates a new location in memory, the stack associated with that location initially contains a single unique tag with pointer ID zero, \( \text{Unique}(0) \), which is the tag of the owner of the location.

## Rules

The following rules are enforced by the Stacked Borrows model to validate memory accesses and the creation of new mutable or immutable references.
Only the rules that are relevant to imem are presented in this section.

***NEW-MUTABLE-REF:***
Any time the program creates a new mutable reference, using `&mut expr`, from some existing pointer value $\text{Pointer}(l,t)$, the referent, first of all this is considered a use of that pointer value (so the model follows **USE-1** below).
Then consider some fresh pointer ID $t'$.
The new reference gets value $\text{Pointer}(l,t')$, and the model pushes $\text{Unique}(t')$ on top of the stack for $l$.

***USE-1:***
Any time the program uses a pointer value $\text{Pointer(l,t)}$ mutably, a tag with pointer ID $t$ must be in the stack for $l$.
If there are other tags above it, pop them, so that the tag with ID $t$ is on top of the stack afterwards.
If $\text{Unique(t)}$ is not in the stack at all, this program has undefined behavior.

The following examples demonstrate how the stack associated with a location changes according to the *NEW-MUTABLE-REF* rule, and how the model validates memory accesses according to the *USE-1* rule:

```Rust
let mut local = 0; // allocates memory l with owner `local` = Pointer(l, 0)
// Stack: [ Unique(0) ]

let x = &mut local; // borrows `local` mutably using NEW-MUTABLE-REF, resulting in `x` = Pointer(l, 1)
// Stack: [ Unique(0), Unique(1) ]

let y = &mut *x; // re-borrows `x` mutably using NEW-MUTABLE-REF, resulting in `y` = Pointer(l, 2)
// Stack: [ Unique(0), Unique(1), Unique(2) ]

*x = 1; // uses `x` mutably via USE-1 rule, which pops everything above `Unique(1)`
// Stack: [ Unique(0), Unique(1) ]

*y = 2; // attempts to use `y` via USE-1, causing error because `Unique(2)` is not in the stack
```

***NEW-SHARED-REF-1:***
Any time the program creates a new immutable reference, using `&expr`, from some existing pointer value $\text{Pointer(l,t)}$, first of all this is considered a read access to that pointer value, so the model follows **READ-1** below.
Then consider some fresh pointer ID $t'$, use $\text{Pointer(l,t')}$ as the value for the shared reference, and add $\text{Shared}(t)$ to the top of the stack for $l$.

***READ-1:***
Any time the program reads a pointer with value $\text{Pointer(l,t)}$, a tag with pointer ID $t$ must exist in the stack for $l$.
Pop items off the stack until all the tags above the tag with ID $t$ are $\text{Shared}(_)$.
If no such tag exists in the stack, the program violates the stack principle.

The following examples demonstrate how the stack associated with a location changes according to the *NEW-MUTABLE-REF* and *NEW-SHARED-REF-1* rules, and how the model validates memory accesses according to *USE-1* and *READ-1* rules:

```Rust
let mut local = 42; // allocates memory l with owner `local` = Pointer(l, 0)
// Stack: [ Unique(0) ]

let x = &mut local; // borrows `local` mutably using NEW-MUTABLE-REF, resulting in `x` = Pointer(l, 1)
// Stack: [ Unique(0), Unique(1) ]

let ir1 = &*x; // borrows `x` immutably using NEW-SHARED-REF-1, resulting in `ir1` = Pointer(l, 2)
// Stack: [ Unique(0), Unique(1), Shared(2) ]

let ir2 = &*x; // borrows `x` immutably again using NEW-SHARED-REF-1, READ-1 on `x` allows `Shared(2)` to stay, resulting in `ir2` = Pointer(l, 3)
// Stack: [ Unique(0), Unique(1), Shared(2), Shared(3) ]

let val = *x; // reads `x` via READ-1, `Shared` tags above `Unique(1)` are allowed to stay
// Stack: [ Unique(0), Unique(1), Shared(2), Shared(3) ]

let val = *ir1; // reads `ir1` via READ-1, `Shared(3)` above `Shared(2)` is allowed to stay
// Stack: [ Unique(0), Unique(1), Shared(2), Shared(3) ]

let val = *ir2; // reads `ir2` via READ-1, `Shared(3)` is at the top
// Stack: [ Unique(0), Unique(1), Shared(2), Shared(3) ]

*x += 17; // uses `x` mutably via USE-1 rule, which pops everything above `Unique(1)` (both shared tags)
// Stack: [ Unique(0), Unique(1) ]

let val = *ir1; // attempts to read `ir1` via READ-1, causing error because `Shared(2)` is not in the stack
// Stack: [ Unique(0), Unique(1) ] -> UNDEFINED BEHAVIOR
```

## imem

The Stacked Borrows model provides a dynamic verification of the static rules enforced by Rust’s borrow checker.
Because it is a dynamic analysis, it is more complete, meaning, some Rust programs that do not violate the Stacked Borrows model and do not exhibit undefined behavior are still rejected by the borrow checker.

imem follows the Stacked Borrows model in two independent directions:

- At [compile time](../imem/memory-management.md): imem enforces static rules that are sound.
  Therefore, accepted programs follow the Stacked Borrows model at runtime.
  However, similar to Rust, these static rules are more restrictive and may reject programs that are in fact valid.

- During [Runtime](../imem/runtime.md):
  imem enforces a subset of the Stacked Borrows model within its memory model to demonstrate that statically accepted programs do not violate the dynamic rules.
  In addition, if a program does not follow the [guidelines](../imem/soundness.md) provided by imem or uses Scala features that break imem’s static guarantees, the dynamic analysis verifies memory accesses at runtime.
