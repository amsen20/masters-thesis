# Memory model

This section explains how Scinear rule enforcement shapes the graph of linear and nonlinear objects at runtime. Because `@HideLinearity` significantly affects the memory model, this discussion is structured into two parts.

For simplicity, Linear types are assumed not to be cast to `Object`.

## With no `@HideLinearity`

### Memory Overview

In the absence of `@HideLinearity`, references to a linear type are not stored within a non-linear type, other than `Option` and `Tuple`, due to the [nonlinear-type-field-rule](/docs/scinear/using-linear-types.md#fields-in-other-types). Additionally, linear values are referenced only by other linear values or by terms that Scinear statically identifies as linear. Meaning, Scinear rules apply to these terms.

Reachable linear objects form a tree rooted in the program environment. The children of this root are the linear terms currently available in the environment, and the remaining nodes are linear values stored within other linear values. Additionally, linear values may refer to nonlinear values. The following figure demonstrates this memory overview:

![Memory model overview](../img/linear-memory-overview.png){: width="700"}

### Why

Operations related to linear values fall into one of the following categories:

* Reading a field or calling a method: This interaction expires the linear term and designates the result of the method or field access as a new term.  
* Creating a linear term: This process results in a new linear term. Any linear terms used as arguments for the type constructor become children of this new term.

Based on the operations, the tree structure is consistently preserved.

## With `@HideLinearity`

The `@HideLinaerity` annotation allows bypassing all Scinear rules. Specifically, wrapping the code that causes the error in a function annotated with `@HideLinaerity` that accepts a linear term as a polymorphic argument converts an error into a warning. This mechanism is similar to Rust's `unsafe` keyword.  
As a result, the use of this annotation makes any object graph configuration possible. Similar to Rust’s unsafe, imem uses this feature to build basic memory management blocks. While these blocks are internally unsafe, they expose a safe interface.