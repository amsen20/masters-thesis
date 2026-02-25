# Capturing Types

Capture checking is a [research project](https://dl.acm.org/doi/full/10.1145/3618003) and a currently [experimental feature](https://docs.scala-lang.org/scala3/reference/experimental/cc.html) in the Scala type system that allows values to track capabilities.
This section explains a simplified version of capture checking.

## Overview

In capture checking, values track capabilities through their types.
In Scala, with this feature enabled, every type `T` is augmented with a *capture set* `{c1, c2, ..., cn}`, where each member of the set is a capability.
Capture checking refers to types with a non-empty capture set as capturing types, written as `T^{c1, c2, ..., cn}`.

In capture checking, a capability is either the universal capability `cap` or a local variable or method or class parameter that has a type with a non-empty capture set.
Therefore, all capabilities are derived from other capabilities, except for `cap`, which is the root capability.
The notation `T^` denotes a type that captures only the universal capability, that is, `T^{cap}`. 

In the following example, `fs` is a capability because its type capture `cap`:

```Scala
class Logger(fs: FileSystem^): ...
```

## Function Types

Capture checking augments function types with capture sets, similar to other types.
A function type `A -> B` denotes a function with an empty capture set, whereas a function type `A => B` denotes a function that captures the universal capability, `cap`.
A function can also have a specific capture set.
For example, `A -> {c, d} B` denotes a function that captures `{c, d}`.

A function captures the capabilities that it refers to in its body, or the capabilities captured by functions that it calls in its body.
Functions with empty capture sets are called pure functions, whereas functions that capture the universal capability are called impure functions.

For example, the `log` method below captures `fs` in its capture set:

```Scala
val fs: FileSystem^ = ...

def log(s: String): Unit = fs.println(s)
```

## Sub-typing and Sub-capturing

A type `T1^C1` is a subtype of `T2^C2`, written as `T1^C1 <: T2^C2`, if `T1 <: T2` and `C1 <: C2`.
A capture set `C1 = {c11, c12, ..., c1n}` sub-captures another capture set `C2 = {c21, c22, ..., c2m}` if, for every capability `c` in `C1`, one of the following conditions holds:

- `c ∈ C2`, or
- the type of `c` has a capture set `C'` such that `C' <: C2`.

As an expected result, every capture set sub-captures any capture set that contains the universal capability, `{cap}`.

## Capture Set Type Parameters

Type parameters can only be instantiated with types that have empty capture sets.
To accept arguments whose types have non-empty capture sets, the argument type must be annotated with `cap` as its capture set.
For example, consider the following function:

```Scala
def f[T](x: T^): ...
```

`f` takes an argument that is an instance of `T` and can capture any capability, because every type `T^C` is a subtype of `T^`, that is, `T^{cap}`.  
Therefore, the argument of `f` is implicitly polymorphic in its capture set.
To make this polymorphism explicit, a [new feature](https://contributors.scala-lang.org/t/experimental-capture-checking-new-syntax-for-explicit-capture-polymorphism/7095) allows programs to declare type parameters that represent capture sets.
For example, the following is a rewrite of `f` with explicit capture set type parameters:

```Scala
def f[T, CS^](x: T^{CS}): ...
```

In the definition of `f` above, `CS^` is a capture set type parameter that can be instantiated with any capture set.

This explicit form gives the program more control over capture set polymorphism.
Because they are explicit, it is possible to enforce specific relationships between these parameters.
In addition, the program can specify lower and upper bounds for them.
The following examples demonstrate how to enforce that `CS^` always includes the `fs` capability:

```Scala
def f[T, CS^ >: {fs}](x : T^{CS}): ...
```

Furthermore, it is possible to use capture sets in a class type argument without capturing them.
This design allows the program to enforce sub-capturing rules explicitly where needed, instead of having these rules applied implicitly in the background.
For example, the following illustrates a stricter restriction on the capture set when it is expressed as a capture-set type parameter rather than as an annotated capture set.

```Scala
class AAnn: ...
class AParam[CS^]: ...

val aAnn: AAnn^{fs} = ...
val bAnn: AAnn^ = ann // ok

val aParam: AParam[{fs}] = ...
val bParam: AParam[{cap}] = aParam // error 
```

In the example above, the `AAnn` instance `aAnn` can be assigned to `bAnn` without any issue, because `{fs} <: {cap}`.
However, the `AParam[{fs}]` instance in `aParam` cannot be assigned to `bParam` because the type parameter is not covariant.

To inform the typer that a function body uses a capability referenced by a capture set type parameter, the function must annotate that type parameter with `@caps.use`.
This annotation is similar to including the capture set type parameter in the function’s own capture set.

```Scala
def f[T, @caps.use CS^](x: T^{CS}): Unit = ...
```

In the above example, the function type is: `f: [T, CS^] =>> [x: T^{CS}] ->{CS} Unit`.

## Escape Checking

Capture checking does not allow certain capture sets to include `cap`.  
One such case is a type parameter.
A type parameter `T` cannot be instantiated with a type that explicitly captures `cap`, for example `S^`.  
However, `T` can be instantiated with `S`, in which case `T^` becomes `S^`.
For example, consider the following:

```Scala
def withFs[T](body: Filesystem^ => T): T = ...

withFs(fs => () => fs.write()) // error
```

At the use site of `withFs`, `T` is instantiated with `() ->{fs} Unit`.
Since `fs` is not defined in the scope of `T`, the typer replaces it with the smallest superset of `{fs}`, which is `cap`.
The process of replacing a capture set with the smallest non-local superset of it is called [avoidance](https://docs.scala-lang.org/scala3/reference/experimental/cc.html#subtyping-and-subcapturing).
As a result, `T` is instantiated with `() ->{cap} Unit`, which captures `cap` and is therefore not allowed.

This mechanism enables a functionality called escape checking.
A function that receives a continuation-passing-style argument can ensure that the argument will not leak the value passed to it, meaning the value is not stored in another object, captured by a closure, or returned after applying the continuation to the value.
This guarantee is achieved by annotating the closure argument with the universal capability.
In the example above, the first argument of `body` is escape-checked.

### Escape Checking Propagation

To ensure that all values derived from an escape-checked argument remain escape-checked, the program must follow specific patterns.  
Two patterns are particularly useful in the context of imem.

First, if a function receives a value that may be escape-checked and intends to propagate escape checking to its return value, the function must take a universally capturing argument, and the return type must capture that argument.

```Scala
def f(x: Object^): String^{x} = ...
```

For example, if an `Object` instance is escape-checked and passed to `f`, the resulting `String` instance is bound to the same scope as the argument `x`.
In other words, the resulting `String` is also escape-checked.

Second, if a class instance may be escape-checked and needs to propagate escape checking to its fields, the class fields must capture `this`.
For example, in the following definition, if an instance of `MyPair` is bound to a scope using escape checking and cannot leak from that scope, then its `first` and `second` fields are also bound to the same scope.

```Scala
class MyPair[T, S](_first: T, _second: S):
  val first: T^{this} = _first
  val second: S^{this} = _second
```

The fields `first` and `second` are not defined directly as constructor arguments because `this` is not available in the scope of the constructor parameters.
Therefore, their types cannot capture `this` at that point.

## Access Control

A use case of explicit capture set type parameter bounds, called [Access Control](http://github.com/scala/scala3/blob/main/docs/_docs/reference/experimental/capture-checking/advanced.md#access-control), ensures that a continuation-passing-style argument is instantiated with a closure that does not use a specific capability.
For example, consider the following:

```Scala
// This is a 'brand' capability that marks access to the filesystem or network
object FileSystem extends caps.Capability
object Network extends caps.Capability

val printerToFs: Printer^{FileSystem} = ...
val printerToNet: Printer^{Network} = ...

// The `body` may refer to (and use) the filesystem, but not the network
def run[C^ <: {FileSystem}](body: () ->{C} Unit): Unit = ...

run(() => printerToFs.write("hey")) // ok
run(() => printerToNet.write("hey")) // error
```

In the example above, any continuation that instantiates `body` cannot use the `Network` capability and may only refer to the `FileSystem` capability.
