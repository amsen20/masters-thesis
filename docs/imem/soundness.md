# Soundness

This section explains why imem’s implementation ensures that the part of program memory managed by imem follows the well-formedness properties defined in the [memory management](./memory-management.md#well-formedness_2) section.
It then discusses loopholes and unwanted API uses, whether they break imem’s guarantees, and the guidelines for avoiding those that do.

## Satisfying Well-formedness Properties

The following lists the memory [well-formedness properties](./memory-management.md#well-formedness_2) in imem and explains how the implementation satisfies each property:

- **Direct Box Uniqueness:**
  The imem implementation of ownership and Scinear linearity rules follows the theoretical definitions of ownership and linearity described in the [memory management section](./memory-management.md#memory-with-imem-references-and-lifetimes).
  The only difference appears in the case of moving, where the *direct-box lemma* proves the preservation of *Direct Box Uniqueness*.

- **No cyclic box:**
  The linearity of the `Box` class ensures that a program has only one access to a box instance.
  In addition, borrow checking guarantees that the program can access either a mutable reference to a box or the box itself, but not both at the same time.
  Together with the fact that boxes are the only source of mutability in imem, these properties ensure that the program cannot use `setBox` to assign a box’s value to itself or to a reference that reaches the same box.
  As a result, the implementation satisfies *No cyclic box*.

- **No dangling references:**
  The imem implementation does not fully satisfy the [theoretical definition](./memory-management.md#well-properties-1) of this property.
  For example, if one `Box` instance reaches another `Box`, the capture set type argument of the reaching box is not always a superset of that of the reached box.
  The *dangling-box-unavailability lemma* shows that, although defining dangling boxes is possible, such boxes become unavailable once the box they reach expires.

- **Borrowing Validity:**
  The [borrowing interfaces](./borrow-checking.md#reference-ownership-management) align with the [borrowing operations](./memory-management.md#new-and-changed-program-needed-operations).
  They satisfy the property by ensuring that a reference resulting from borrowing does not outlive the original box.

- **Reaching Properties**:
  The borrow-checking [goals sub-section](./borrow-checking.md#goals) explains in detail how the implementation satisfies the reaching properties, namely *Box reaching Reference*, *Mutable Reference reaching Immutable Reference*, *Mutable Reference reaching Mutable Reference*, and *Immutable Reference not reaching Mutable Reference*.
  The implementation enforces these properties through [Box and Reference Holding](./borrow-checking.md#box-and-reference-holding), [Reference Owner Aggregation](./borrow-checking.md#reference-owner-aggregation), and [Static Permission Management](./borrow-checking.md#static-permission-management-of-operations).

### Supporting Lemmas

#### Implementation Definition of Availability

In the imem implementation, the definition of availability differs slightly from the definition in the [memory management section](./memory-management.md#additional-definitions-1).
The implementation defines availability based on the entire box or reference type, rather than only on its direct owner set.

***Box and Reference Availability***
At each point during program execution, a box or a reference is available if its type does not include any type parameter, direct or nested, instantiated with a capture set that contains an expired linear capability.

***Variable Availability:***
A variable is available in the program scope at a point during execution if referring to the variable in an expression does not at that point produce a compiler error.
In the imem context, this means:

- The variable is defined in one of the enclosing scopes.
- If the variable has a linear type, it has not expired.
- If the declared type of the variable includes type parameters instantiated with capture sets, any linear capability in those capture sets has not expired.

#### Lemmas

***dangling-box-unavailability-lemma:***
If a value \( v : V \) reaches a box instance \( b' : \text{Box}[T', Owner'] \) and a lifetime capability \( lf \in Owner' \) expires, then \( v \) becomes unavailable.

**Proof.**
Let \(v: V\) be a value that reaches a box instance \(b': \text{Box}[T', Owner']\).
The value \(v\) can be either a box \(b: \text{Box}[T, Owner]\) or a linear value \(l: L\).
If a lifetime capability \(lf\) is in \(Owner'\), \(lf \in Owner'\), the following proves that \(lf\) occurs in at least one capture set instantiation of a type parameter of \(V\):

If a box \(b\) reaches another box \(b'\), then either \(b = b'\), or \(deref(b)\) reaches \(b'\).
If a linear value \(l\) reaches a box \(b'\), then at least one field value \(f: F\) reaches \(b'\).

The proof uses induction on the number of reachability operations needed to obtain \(b'\) from \(v\).
Let \(n \ge 0\) be such that there exists a sequence \(Op_1, \ldots, Op_n\) with \(Op_1 \circ \cdots \circ Op_n (v) = b'\).

**Induction statement for \(n\) operations:**
If \(v: V\) reaches \(b': \text{Box}[T', Owner']\) using \(n\) operations and \(lf \in Owner'\), then \(lf\) occurs in at least one capture-set instantiation of a type parameter of \(V\).

**Base case (\(n = 0\)):**
With zero operations, \(v = b'\).
Therefore, \(V\) is \(\text{Box}[T', Owner']\), and \(lf \in Owner'\) appears in the instantiation of a type parameter of \(V\).

**Inductive step (\(n \ge 1\)):**
The inductive step depends on the first operation from \(v\).
In each case, the induction-hypothesis occurrence of \(lf\) in the intermediate value’s type is propagated back to a type-parameter instantiation of \(V\).

- If \(v\) is a box \(b: \text{Box}[T, Owner]\), then \(deref(b): T\) reaches \(b'\).
  Based on the induction hypothesis, there exists a type parameter \(p\) of \(T\) that is instantiated with a capture set containing \(lf\).
  Since \(T\) is a type parameter of \(\text{Box}[T, Owner]\), the instantiation of \(T\) includes the instantiation of \(p\), and therefore includes \(lf\).

- If \(v\) is a linear value \(l: L\), then some field value \(f: F\) reaches \(b'\).
  By the induction hypothesis there exists a type parameter \(p\) of \(F\) that is instantiated with a capture set containing \(lf\).
  Each type parameter of \(F\) is instantiated either with types defined in \(L\) or with types built from type parameters of \(L\).
  Since linear classes do not allow nested type definitions, there exists a type parameter of \(L\) that \(p\) is instantiated with, and this instantiation contains a capture set that includes \(lf\).

By induction, if a value \( v : V \) reaches a box instance \( b' : \text{Box}[T', Owner'] \) and \( Owner' \) includes a lifetime \( lf \), then the capability \( lf \) appears in a capture-set instantiation of a type parameter of \( V \).  
Therefore, if \( lf \) expires, the value \( v : V \) becomes unavailable, proving the lemma.

***direct-box-lemma:***
Assume that every resource has exactly one direct box.
After a `moveBox` or `derefForMoving` operation, every resource still has exactly one direct box.

**Proof.** The proof consists of two parts, depending on the operation.

**`moveBox`:**
Assume that the program applies `moveBox` to a box \(b : \text{Box}[T, Owner]\).
Since `Box` is a linear type, the program cannot reach \(b\) after \(b\) is passed to `moveBox`.
The operation returns a box \(b' : \text{Box}[T, Owner']\), which is a direct box for the resource of \(b\).
Before the operation, \(b\) is the only reachable direct box for that resource.
After the operation, \(b\) is unreachable and \(b'\) is reachable.
Therefore, the resource has exactly one reachable direct box after the operation.

**`derefForMoving`:**
Assume that the program applies `derefForMoving` to a box \(b : \text{Box}[T, Owner]\).
Since `Box` is a linear type, the program cannot reach \(b\) after it is passed to `derefForMoving`, and the program also cannot reach \(b\) during the `moveAction` continuation.

During `moveAction`, the value \(t = deref(b)\) is passed to the continuation as an [escape-checked](../background/capturing-types.md#escape-checking) argument.
The value \(t\) is not a resource because there is no available reachable box pointing to it.
Moreover, the only box that provides access to \(t\) is \(b\), and \(b\) is inaccessible inside `moveAction`.
Also, because the `newBox` function requires a resource argument with an empty capture set, the program, inside `moveAction`, cannot create a new box that turns \(t\) into a resource again.

After `moveAction`, since \(t\) was an escape-checked argument in `moveAction`; the program can only access \(t\) after `moveAction` if `moveAction` returns \(t\).
In that case, \(t\) is still not a resource, and the statement of the lemma continues to hold.

## Possible Loopholes and Guidelines

The following subsections list possible loopholes or unwanted cases and discuss whether a valid program can exhibit them and, if so, following what guidelines would avoid reaching any memory state that is not well-formed.

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

However, imem aims to provide safe mutability through static rules enforcing the [Stacked Borrows Model](../background/stacked-borrows.md).
The only mutations that imem manages are calls to the `swapBox` and `setBox` functions on imem boxes.
Other sources of mutation are outside the control of imem, and when a program includes them, imem cannot guarantee their safety.

### Escape Checking

imem uses [escape-checking](../background/capturing-types.md#escape-checking) to prevent resources from leaking outside their intended scope.
As discussed in the [escape-checking propagation](../background/capturing-types.md#propagating-to-fields) section, for escape-checking to propagate from a class instance to its fields, each field must either be a capability or have a type that captures `this`.
Otherwise, when an instance is subject to escape-checking, its fields are not bound to the same scope as the instance.

This limitation is particularly important when the resource in a box is a linear class with multiple fields.
This is illustrated by the following example:

```Scala
class LeakyPair(_first: LinearInt, _second: LinearInt) extends scinear.Linear:
  val first: LinearInt = _first // leaky because does not capture `{this}`
  val second: LinearInt = _second // leaky because does not capture `{this}`
end LeakyPair

val lf = Lifetime[{ctx}]()
val box: Box[LeakyPair, {lf}] = newBox(LeakyPair(LinearInt(1), LinearInt(2)))
val (immutRef, boxHolder) = borrowImmutBox[LeakyPair, {lf}, {ctx}, lf.Key, lf.Owners, WC, MC](box)

val leakedField = read[LeakyPair, lf.Owners, LinearInt, {ctx}, WC, MC](
  immutRef,
  resource =>
    resource.first // ! leaking the field of the resource of the box
)
```

If the class definition does not follow the escape-checking propagation rules, a program may leak the class fields.
imem assumes that every linear class that is a resource in a box satisfies these requirements.

### imem Components

There are several possible workarounds related to imem’s components.
imem addresses some of these by special features in the imem implementation that are specific to each loophole.
At the same time, there are other workarounds that imem does not resolve and are instead left to guidelines.
The following section lists these cases.

#### Internal Components

imem doesn't allow the program to have access to the internal components by:

- Making `UnsafeRef` and `InternalRef` private to the imem package.
- Marking all fields of `Box`, `ImmutRef` and `MutRef` class as private.

In this way, the program is not able to neither import internal components, instantiate them, nor access them through publicly accessible classes in imem.

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

##### Resource

The `Box` class constructor is private, which means that a program can create a box only through the imem interfaces.
These interfaces take a resource as input and return a box that points to that resource.
imem assumes that the only Scala reference to the resource is the one passed to the box constructor.
In other words, the program does not store the Scala reference of the resource, or any object that can reach that resource, elsewhere in memory.
In the following example, the program creates two boxes that refer to the same resource:

```Scala
class LinearArray(_first: Array[Int]) extends scinear.Linear:
  val first: Array[Int]^{this} = _first
end LinearArray

val lf = Lifetime[{ctx}]
val arr: Array[Int] = Array(1, 2, 3)

val arrBox1: Box[LinearArray, {lf}] = newBox(LinearArray(arr))
val arrBox2: Box[LinearArray, {lf}] = newBox(LinearArray(arr))
//! both resource of the `arrBox1` and `arrBox2` point to the same array
```

imem controls only access to a resource in a box, not the resource reaching memory.
This means that the rest of the memory that the resource reaches is not controlled by imem, unless that memory is itself another box or a box’s resource.
Therefore, if a program creates multiple resources that point to the same underlying object via Scala references, imem regulates access to those resources, not to the object they point to.
Managing access to the shared object itself remains the program's responsibility.

It is important to note that imem’s static guarantees no longer apply when a program reaches the non-linear part of memory.
A resource in a box must be a linear class, and access to linear instances is statically controlled according to Scinear rules.
If the fields of the resource class are also linear, Scinear statically manages access to the memory associated with those fields as well, and this property propagates transitively.
For this reason, imem recommends keeping memory linear as much as possible.

One important guideline for linear memory is to replace direct linear fields with box fields that point to linear classes.
For example, in the following `LinearBinTree` class, the fields have type `LinearBinTree`:

```Scala
class LinearBinTree(
	_left: Option[LinearBinTree],
	_right: Option[LinearBinTree]
) extends scinear.Linear:
	val left: Option[LinearBinTree]^{this} = _left
	val right: Option[LinearBinTree]^{this} = _right
end LinearBinTree
```

According to imem’s guarantees, this implementation is correct.
However, when the program has access to the children of a `LinearBinTree` instance, it cannot borrow them immutably or mutably.
It also cannot reference both of a node's children simultaneously or iterate over the tree without consuming the tree and destructing and reconstructing it during the traversal.

```Scala
class LinearBinTree(
	_left: Option[LinearBinTree],
	_right: Option[LinearBinTree]
) extends scinear.Linear:
	val left: Option[LinearBinTree]^{this} = _left
	val right: Option[LinearBinTree]^{this} = _right
end LinearBinTree

// a three node binary tree
val leftLeaf = LinearBinTree(None, None)
val rightLeaf = LinearBinTree(None, None)
val rootNode = LinearBinTree(Some(leftLeaf), Some(rightLeaf))

val lf = Lifetime[{ctx}]()
val treeBox = newBox[LinearBinTree, {lf}](rootNode)

val (treeRef, treeHolder) = borrowImmutBox[LinearBinTree, {lf}, {ctx}, lf.Key, lf.Owners, WC, MC](treeBox)

read[LinearBinTree, lf.Owners, Unit, {ctx}, WC, MC](
  treeRef,
  root =>
    root.left // ! Accessing this node is only possible
              // ! in here, the program cannot have a
              // ! reference to this node.
    ()
)
```

The following example shows that when a `LinearBinTree` class refers to its children through `Box` instances, it is possible to hold an immutable reference to both children at the same time.

```Scala
class LinearBinTree[O^](
  _left: Box[Option[LinearBinTree[O]], O],
  _right: Box[Option[LinearBinTree[O]], O]
) extends scinear.Linear:
  val left: Box[Option[LinearBinTree[O]], O]^{this} = _left
  val right: Box[Option[LinearBinTree[O]], O]^{this} = _right
end LinearBinTree

val lf = Lifetime[{ctx}]()

val leftLeaf = LinearBinTree[{lf}](newBox(None), newBox(None))
val rightLeaf = LinearBinTree[{lf}](newBox(None), newBox(None))
val rootNode = LinearBinTree[{lf}](
  newBox(Some(leftLeaf)),
  newBox(Some(rightLeaf))
)
val treeBox = newBox[LinearBinTree[{lf}], {lf}](rootNode)

val lfRef = Lifetime[{ctx}]()
val (rootImmutRef, rootHolder) = borrowImmutBox[LinearBinTree[{lf}], {lf}, {ctx}, lfRef.Key, lfRef.Owners, {WC}, {MC}](treeBox)
val leftImmutRef = read[LinearBinTree[{lf}], lfRef.Owners, ImmutRef[Option[LinearBinTree[{lf}]], {lf, ctx}], {ctx}, {WC}, {MC}](
  rootImmutRef,
  ctx ?=> root =>
    val (leftImmutRef, leftHolder) = borrowImmutBox[Option[LinearBinTree[{lf}]], {lf}, {ctx}, NeverUsableKey, {ctx}, {WC}, {MC}](root.left)
    leftHolder.consume()
    leftImmutRef
)

val rightImmutRef = read[LinearBinTree[{lf}], lfRef.Owners, ImmutRef[Option[LinearBinTree[{lf}]], {lf, ctx}], {ctx}, {WC}, {MC}](
	rootImmutRef,
	ctx ?=> root =>
		val (rightImmutRef, rightHolder) = borrowImmutBox[Option[LinearBinTree[{lf}]], {lf}, {ctx}, NeverUsableKey, {ctx}, {WC}, {MC}](root.right)
		rightHolder.consume()
		rightImmutRef
)

leftImmutRef // the program has a reference to the left child of the root
rightImmutRef // the program has a reference to the right child of the root
rootImmutRef // the program has a reference to the root
```

##### Owner set

In imem, the `Box` class has a capture set type parameter `Owner^`, which the program instantiates with the set of lifetimes that should outlive a box instance.
imem does not statically enforce that all capabilities in this capture set are lifetime capabilities, because current Scala features do not allow such enforcement.

Although it does not seem convenient to include a capability that is not a `Lifetime` instance in the owner set of a `Box`, `ImmutRef`, or `MutRef`, this does not hurt imem's soundness.
As a capture set becomes larger, capture checking applies more restrictions, which preserves the soundness of imem.

### Borrow Checking

#### Borrowing New Owners

When the program borrows a box or a reference, either immutably or mutably, through any imem interface, the function requires a new owner capture set type parameter, `newOwner^`, and the corresponding lifetime key, `newOwnerKey`, for that owner set.
imem assumes that the program instantiates these parameters using one of the following two ways:

- **A lifetime instance with its `Owners` and `Key` type members:**
  This option enables the program to get back the source of the borrowing.
  In this case, the program can unlock the value holder only by using an instance of the associated lifetime key, which causes the lifetime to expire.

- **`NeverUsableKey` with any lifetime set:**
  This option is useful when the program no longer needs the source of the borrow and only requires the borrowed reference for use.
  Because `NeverUsableKey` is an opaque type, the program cannot create instances of it, and therefore cannot unlock the value holder.

There is no static enforcement that restricts valid programs to these two cases.
This means that a program can misuse these parameters and break imem’s guarantees:

```Scala
val lf = Lifetime[{ctx}]()
val box: Box[LinearInt, {lf}] = newBox(LinearInt(42))

val (immutRef, holder) = borrowImmutBox[LinearInt, {lf}, {ctx}, Object, lf.Owners, WC, MC](box) // ! borrowing using `Object` as the key type

// unlocking the holder using an instance of `Object`
val retrievedBox = unlockHolder(Object(), holder)

immutRef // ! the program has access to the reference
retrievedBox // ! the program has access to the box as well
```

This constraint is not enforced statically because there is no way to use a path as a type parameter and require other type parameters to be path-dependent type members of that parameter.
In addition, since the `Key` type member of `Lifetime` is a path-dependent opaque type, it is not possible to place type bounds on type parameters that may be instantiated with it.
The [future work](../conclusion/future-works.md) section proposes an approach that would make misuse of these interfaces more explicit.

#### Context

##### Write and Move Capabilities

The `Context` constructor is private, and `withImem` is the only imem interface that provides a `Context` instance to a program.
Because the `WC^` and `MC^` type parameters of `Context`, which represent write and move capabilities, are neither covariant nor contravariant, a program cannot pass around the `Context` instance provided to it as a `Context` with different capture sets for these parameters.
As a result, the write and move capability sets remain consistent and identical throughout the entire program.

##### Owner Aggregation

[Context owner aggregation](./borrow-checking.md#context-owner-aggregation) uses the capture-set annotation of the `Context` capturing type, `Context[WC, MC]^{ctxOwner}`, to represent the set of accumulated owners that are in scope.
Unlike `WC^` and `MC^`, the `ctxOwner^` capture set is not a type parameter of `Context`, and therefore it is covariant.
As a result, a program can assign a value of type `Context[WC, MC]^{A}` to a value of type `Context[WC, MC]^{B}` when `A <: B`.

This behavior is safe as long as the new capture set still includes the relevant lifetime capabilities.
However, because the universal capability `cap` is a superset of all capture sets, a program can assign a context of type `Context[WC, MC]^{ctxOwner}` to one of type `Context[WC, MC]^{cap}`.
In this case, lifetime expiration no longer limits the context and the references borrowed using that context, which breaks imem rules.

imem assumes that programs do not alter the accumulated owners of a context.
Due to heavy limitations on type capturing the universal capability, `cap`, I was unable to find a concrete example in which this behavior breaks imem’s rules without also causing compile-time errors.
