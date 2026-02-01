# Soundness

This section explains why imem’s implementation ensures that the part of program memory managed by imem follows the well-formedness properties defined in the [memory management](./memory-management.md#well-formedness-2) section.
It then discusses loopholes and unwanted API uses, whether they break imem’s guarantees, and the guidelines for avoiding those that do.

## Satisfying Well-formedness Properties

TODO:

## Possible Loopholes and Guidelines

The following subsections list possible loopholes or unwanted cases and discusses whether a valid program can exhibit them and, if so, following what guidelines would avoid reaching any memory state that is not well-formed.

### General

In general, any use of `asInstanceOf` breaks static type safety and, as a result, all guarantees imem provides.
Similarly, programs that include reflection can manipulate types at runtime, which breaks all imem's static enforcements.
For these features, the guideline is to either avoid their use in an imem program or use them with full awareness of their effects on imem.
imem assumes that programs do not use these features.

### Linearity

imem bases its ownership and borrow checking on Scinear linearity rules.
As a result, any program that does not follow these rules breaks the guarantees provided by imem.
This includes programs that upcast linear types to non-linear types, such as `Object`, or that use `@HideLinearity`.
Such programs may lead to incorrect memory states and, transitively, to unsafe behavior.

### Mutability

imem assumes that boxes are the only source of mutation.
In Scala, a class field or a variable can be mutable when it is defined using `var`.
Because imem is not a compiler plugin and does not parse the program that uses it, it cannot determine whether `var`s are present.

However, imem aims to provide safe mutability through static rules enforcing [Stacked Borrows Model](../background/stacked-borrows.md).
The only mutations that imem manages are calls to the `swapBox` and `setBox` functions on imem boxes.
Other sources of mutation are outside the control of imem, and when a program includes them, imem cannot guarantee their safety.

### Escape Checking

imem uses [escape-checking](../background/capturing-types.md#escape-checking) to prevent resources from leaking outside their intended scope.
As discussed in the [escape-checking propagation](../background/capturing-types.md#propagating-to-fields), for escape-checking to propagate from a class instance to its fields, each field must either be a capability or have a type that captures `this`.
Otherwise, when an instance is subject to escape-checking, its fields are not bound to the same scope as the instance.

This limitation is particularly important when a box’s resource is a linear class with multiple fields.
As the following example illustrates:

```Scala
TODO: A LINEAR CLASS WITH LINEAR FIELDS AND BEING A BOX'S RESOURCE. WHEN THE PROGRAM ACCESS THE RESOURCE IT CAN LEAK THE FIELDS.
```

If the class definition does not follow the escape-checking propagation rules, a program may leak the class fields.
imem assumes that every linear class that is a box’s resource satisfies these requirements.

### imem Components

There are several possible workarounds related to imem’s components.
imem addresses some of these by special features in the imem implementation that are specific to each loophole.
At the same time, there are other workarounds that imem does not resolve and are instead left to guidelines.
The following section lists these cases.

#### Internal Components

imem doesn't allow the program to have access to the internal components by:

- Making `UnsafeRef` and `InternalRef` private to imem package.
- Marking all fields of `Box`, `ImmutRef` and `MutRef` class as private.

In this way the program is not able to neither import internal components, instantiate them, nor access them through publicly accessible classes in imem.

#### Lifetime

In imem, a [Lifetime](../imem/ownership.md#lifetime) represents a static, continuous region of a program.
This region starts at the lifetime definition and ends at the point where the program uses it.
Therefore, a program that uses imem should not pass a lifetime around, except to `unlock`, nor return or copy it.
All such operations are a use of the lifetime, which causes every `Box`, `ValueHolder`, `ImmutRef`, and `MutRef` associated with that lifetime to expire at the point of use.

Another consideration is that restricting lifetime usage to a continuation-passing-style argument, similar to how resources are accessed in the `read` and `write` functions, would simplify the model.
In this approach, the continuous part of a program would correspond to a scope, which is more convenient for static rule enforcement.
However, this restriction would imply that, when a reference includes two lifetimes, one lifetime must start before the other and end after it, creating an indirect dependency between the two lifetimes.
Such a constraint would make imem less expressive and would prevent the use of intermediate references between two lifetimes without forcing one lifetime to dominate the other.

Finally, the lifetime `Key` type is an opaque path-dependent type.
This means that the only way to obtain a key instance for a lifetime, for example `lf`, is by calling `lf.getKey()`.
Calling `lf.getKey()` is a use of the linear capability `lf`, which causes all references that include `lf` in their lifetime set to expire.
Therefore, it is not possible to leak or pass around a lifetime key while the lifetime remains unexpired.

#### Box

------------------------------------------------------------------------------------------------

<!-- TODO: Fix this -->
Also, `WC^` and `MC^` has to be type parameters to 
because the program has access to only one instance of `Context` that is provided to it by imem.
Furthermore, The program is not able to create an instance of `Context`.
So each function has to have two type parameters that the instantiate the `Context` type parameters.

<!-- TODO: Explain `NeverUsableKey` -->
