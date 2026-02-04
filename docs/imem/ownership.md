# Ownership

This section explains how imem statically manages ownership of boxes and resources.

## Ownership Goals

Ownership management in imem's implementation aims to satisfy the following [properties](./memory-management.md#properties_1) of a well-formed imem memory:

- ***No dangling boxes:***
  This property is a subset of the [*No dangling references*](./memory-management.md#properties_1) well-formedness property.
  This subset states that, during execution, all boxes reachable from available variables are alive.
  The combination of ownership and borrow checking derives the full *No dangling references* property, and the details are described in the [soundness section](./soundness.md#satisfying-well-formedness-properties).

- [***Direct Box Uniqueness***](./memory-management.md#properties_1)

## Lifetime

In imem implementation, `Lifetime` class that implements the [lifetime](./memory-management.md#memory-with-imem-references-and-lifetimes), is a linear capability class whose instances own boxes, references, and other lifetimes.
A lifetime capability owns a box, reference, or another lifetime if their types include a type parameter instantiated with a capture set that includes the lifetime capability.

This ownership relation is many-to-many.
That is, multiple lifetime instances can own a box, a reference, or another lifetime, and a single lifetime instance can own multiple boxes, references, or lifetimes.

The program section between a lifetime capability definition and its usage is called the lifetime’s region, where the program can refer to entities owned by the instance.
Those entities become out of reach of the program, also known as unavailable, after the lifetime region ends.

```Scala3
class Lifetime[OwnersInput^] extends scinear.Linear, caps.Capability:
  type Owners^ = {OwnersInput, this}
  ... // rest of the implementation
end Lifetime
```

The `Lifetime` class takes the `OwnersInput^` capture-set type parameter, which represents the lifetime capabilities that own this lifetime and therefore outlive it.
The class also includes an `Owners^` type member, which denotes the list of the lifetime’s owners together with the lifetime capability itself.
The following illustrates how to define a lifetime and use it:
```Scala
TODO: A LIFETIME DEFINITION, USAGE, AND THE SCOPE
```
Lifetimes can also be structured in a nested manner:
```Scala
TODO: TWO NESTED LIFETIMES
```
The program can also define and use lifetimes intertwined as long as they do not own each other:
```
TODO: TWO INTERTWINED LIFETIMES
```

## Box Ownership

Unlike in Rust, boxes in imem do not own themselves; instead, they are owned by lifetimes.
From the imem perspective, boxes are linear instances whose resources can be accessed by converting them into immutable or mutable references through borrowing.
On the other hand, as in Rust, the program can change a box’s owner by moving it.

### Creating Boxes

The following is the `Box` class definition:
```Scala
class Box[T, @caps.use Owner^](
  private [imem] val tag: InternalRef[T]#Tag,
  private [imem] val internalRef: InternalRef[T]
) extends scinear.Linear
```

In addition to the fields related to the internal reference, it includes a capture-set type parameter `Owner^`, which can be instantiated with a set of lifetime capabilities.

imem provides the following function for creating a new box:

```Scala
def newBox[@scinear.HideLinearity T, Owner^](resource: T): Box[T, Owner] = ...
```

The `newBox` function takes a resource instance as an argument and a capture set of owners as type arguments, and it returns a box that holds the given resource and is owned by the set of owners specified by the `Owner^` type argument.

Note that the program supplies the resource, and imem assumes that the resource is not copied or stored elsewhere.
If the resource is linear, such as an instance of `Box`, the Scinear plugin statically enforces that the resource value is reachable only from the expression passed to `newBox`.
However, the resource may be non-linear.
In that case, it is the program’s responsibility to handle any side effects that arise if the resource has additional aliases.

```Scala
TODO: AN EXAMPLE OF CREATING A SIMPLE BOX WITH ONE LIFETIME, THEN WITH TWO.
```

As with other linear types, boxes can also appear as fields of other linear classes:

```Scala
TODO: AN EXAMPLE OF A BOX AS A FIELD OF A CLASS
```

The primary way to use boxes is to define data-structure classes as linear, but instead of storing them directly as fields within one another, to store a `Box` that wraps the type.
This approach simplifies working with the data structure compared to an entirely linear implementation, while also guaranteeing safe memory management in contrast to a non-linear version.

```Scala
TODO: SIMPLE LINKED LIST SIMILAR TO THE LINEAR CHAPTER BUT WITH BOXES
```

The [Evaluation](../evaluation/index.md) chapter details more about implementing a linked list using imem. 


### Type Checking Dangling Boxes

Boxes are useful when they are nested, either directly, `Box[Box[...], ...]`, when one box's resource is another box, or indirectly when the resource is a class that has boxes as fields.

```Scala
TODO: FOR ALL THE CASES BELOW, DEFINE TWO LIFETIME lf1, AND lf2 THAT IT IS NOT SHOWN WHEN THEY ARE USED.
TODO: A NESTED DIRECTLY BOX WITH SAME OWNERS
TODO: A NESTED DIRECTLY BOX WITH DIFFERENT OWNERS
TODO: AN INDIRECT NESTED BOX WITH DIFFERENT OWNERS
```

As illustrated in the example above, nested boxes may have different owners, which means that an inner box can be owned by a different set of lifetimes than the outer box. This situation may appear to allow the owners of the inner box to expire before those of the outer box, which would result in a dangling box:

```Scala
TODO: THE LAST EXAMPLE OF TWO DIRECTED NESTED BOXES BUT THE INNER BOX LIFETIME IS USED
```

Although this situation is possible, as shown above, it does not lead to dangling boxes.
Unlike the [*No dangling references*](./memory-management.md#properties_1) property defined in memory management, the imem implementation does not enforce that a box reaching another box, either directly or indirectly, has an owner lifetime that is a superset of the reached box’s owner set.
However, the [*dangling-box-unavailability-lemma*](./soundness.md#satisfying-well-formedness-properties) proves that the implementation ensures a box becomes unavailable when it can reach a box with an expired lifetime in its owner set.

As a demonstration, in the previous example, the outer box does not mention the expired lifetime capability directly.
However, Scinear still rejects any attempt to refer to the outer box after that the lifetime capability expires:

```Scala
TODO: THE LAST EXAMPLE OF TWO DIRECTED NESTED BOXES BUT THE INNER BOX LIFETIME IS USED AND SHOWED THAT THE OUTER BOX CANNOT BE MENTIONED
```

### Modifying Boxes

The [borrow checking section](./borrow-checking.md) explains how to borrow a box to access its resource, but, at the same time, imem provides two interfaces that allow direct modification of a box.

One function is for setting a box's resource:

```Scala
def setBox[@scinear.HideLinearity T, Owner^, ...](self: Box[T, Owner]^, resource: T)(...): Box[T, Owner]^{self} = ...
```

Because `Box` is linear, the `setBox` function returns a new `Box` reference that points to the same `Box` instance passed as the `self` argument.

Two aspects of the function signature make `setBox` compatible with other `imem` interfaces.
First, the `self` argument uses a capturing type with the annotated capture set `{cap}`.
The type `Box[T, Owner]^` is equivalent to `Box[T, Owner]^{cap}`.
This choice allows an [escape-checked](../background/capturing-types.md#escape-checking) box to be passed to the function.
The [resource ownership](#resource-ownership) subsection explains why this property is essential.

Second, the returned `Box` captures `self`, as indicated by the type `Box[T, Owner]^{self}`.
Therefore, if the `self` is [escape-checked](../background/capturing-types.md#escape-checking), then the returned reference is also escape-checked, meaning it remains bound to the same scope as the input argument.

The other function is for swapping two boxes’ resources:

```Scala
def swapBox[@scinear.HideLinearity T, @caps.use Owner^, @caps.use OtherOwner^, ...](
  self: Box[T, Owner]^, other: Box[T, OtherOwner]^
)(...): (Box[T, Owner]^{self}, Box[T, OtherOwner]^{other}) = ...
```

Like `setBox`, `swapBox` returns two new `Box` instances that point to the same `Box` instances passed as `self` and `other`.

Both arguments use the `{cap}` capturing annotation, which allows the program to pass escape-checked box variables to the function safely.
In addition, each returned reference captures its corresponding argument, so the results remain bound to the same scopes as the original references.

### Moving Boxes

Moving a box means changing its owner.

The main reason to change ownership is when a box separates from or joins other boxes.
For example, when an element is removed from a list, its lifetime is no longer tied to the list.
If both are represented as boxes, they should then have different owners.

It is important to note that a `Box` instance can change its owner.
However, the owner associated with a variable that refers to the box instance is part of the variable’s type and does not change.

The following function in imem enables moving a Box:

```Scala
def moveBox[@scinear.HideLinearity T, Owner^, NewOwner^, ...](
  self: Box[T, Owner]^
)(...): Box[T, NewOwner] = ...
```

The `moveBox` function takes the current and the new owner as type parameters and returns a new `Box` instance with the updated owner.

```Scala
TODO: AN EXAMPLE OF MOVING THAT ESCAPES THE MOVED BOX
```

As with `setBox`, the `self` parameter captures the universal capability, but the returned box does not.
Otherwise, the box instance resulting from the move would be unable to escape the scope of the original reference, which would make it impractical.

A plain move operation can violate the *Direct box uniqueness* goal when the program applies it to a box that is reached through `read` or `write` operations on references.
This would result in the moved box instance remains reachable from available variables after the move.
The operation rules in [borrow checking](./borrow-checking.md#operation-rules) detail more about this problem.
To prevent this situation, imem enforces restrictions on move operations.

By relying on [Static Permission Management of Operations](./borrow-checking.md#static-permission-management-of-operations), as described in the [borrow checking section](./borrow-checking.md), imem doesn't allow moving inside `read` or `write` operations.
As a result, when a box instance is reachable through other boxes, imem provides a dedicated dereferencing operation designed specifically for moving, called `derefForMoving`.

imem’s interface for dereferencing a box for moving is as follows:

```Scala
def derefForMoving[@scinear.HideLinearity T, ..., @scinear.HideLinearity S, ...](
  self: Box[T, Owner]^,
  moveAction: ... ?->{...} T^ ->{...} S
)(...): S =
```

The `derefForMoving` receives a `self: Box`, which is the box that the operation dereferences.
The function also receives a `moveAction`, which is a continuation-passing-style argument.
The `moveAction` receives the resource, escape-checked, as an argument.

Typically, `moveAction` traverses the given resource to reach other boxes.
It then either calls `derefForMoving` on those boxes to explore their resources or calls `moveBox` to set their owners to a new set of lifetimes.

`derefForMoving` only returns the result of `moveAction`.
Therefore, the box instance passed as the `self` argument expires after `derefForMoving`, and there is no way to recover it.
This behavior is intentional, and the [Uniqueness of Direct Box subsection](#uniqueness-of-direct-box-during-moving) explains how it preserves the *Direct box uniqueness* goal.

TODO: A DIAGRAM OF DANGLING REFERENCE WHEN MOVING

In imem, moving is a sequence of operations applied to the values referenced by available variables:

- Dereference a box using `derefForMoving`
- Decompose the linear value into its fields
- Move the box to a new owner

TODO: A DIAGRAM FOR EACH STEP OF THE MOVING PROCESS

Here is an example of moving the inner box of a nested box `Box[Box[Int, {lf1}], {lf1}]` to `lf2`:

```Scala
TODO: AN EXAMPLE OF MOVING `Box[Box[Int, {lf1}]{lf1}]` to `lf2`
```

The [*direct-box-lemma*](./soundness.md#satisfying-well-formedness-properties) proves that moving interfaces in the imem memory management implementation preserves the [*Direct Box Uniqueness*](./memory-management.md#properties_1) property.

## Resource Ownership

Resources can have any type and do not necessarily include a capture-set parameter for owners.
In imem, resources, together with the unsafe and internal references that hold them, are owned by the same set of lifetime capabilities that own the resource's direct box.

Imem enforces this ownership by allowing the program to access the resource only through escape-checked interfaces.

The program can access a resource through the `read`, `write`, and `derefForMoving` interfaces.
In each case, the continuation that receives access to the resource has a type of the form `... T^ ->{...} S ...`, which indicates that the resource `T` is [escape-checked](../background/capturing-types.md).
As a result, the program can use the resource only within the `writeAction`, `readAction`, or `moveAction` scope and cannot leak it, return it, or store it elsewhere.

Imem can express complex data structures when a box's resource is another `Box`, `ImmutRef`, or `MutRef`.
To support this, all imem's interfaces accept an argument with a capturing type that includes the universal capability, such as `self: Box[...]^`, `self: MutRef[...]^`, and `self: ImmutRef[...]^`.
In addition, when a provided function may return a `Box`, `MutRef`, or `ImmutRef` that points to the same internal reference as its argument, the return type captures that argument.
As a result, the returned value cannot be leaked and is bound to the same scope as the argument.

For example, in `setBox`, the returned type is `Box[T, Owner]^{self}`.
In `swapBox`, both arguments, `self: Box[T, Owner]^` and `other: Box[T, OtherOwner]^`, capture the universal capability and the return types `(Box[T, Owner]^{self}, Box[T, OtherOwner]^{other})` captures each argument respectively.

## Memory Overview

Based on *Direct box uniqueness* and *No dangling boxes*, the object graph of boxes that are reachable from available variables forms a tree structure at runtime.
The following diagram illustrates this tree:

TODO: A SIMPLE DIAGRAM OF BOXES FORMING A TREE
