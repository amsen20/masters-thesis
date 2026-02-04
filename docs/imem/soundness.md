# Soundness

This section explains why imem’s implementation ensures that the part of program memory managed by imem follows the well-formedness properties defined in the [memory management](./memory-management.md#well-formedness-2) section.
It then discusses loopholes and unwanted API uses, whether they break imem’s guarantees, and the guidelines for avoiding those that do.

## Satisfying Well-formedness Properties

------------------------------------------------------------------------------

TODO: START BY MAPPING EACH WELL FORMED NESS PROPERTY TO EXACTLY WHERE IT IS IMPLEMENTED

TODO: DESCRIBE THIS

- ***No dangling boxes:***
  Ownership ensures a subset of the [*No dangling references*](./memory-management.md#properties_1) well-formedness property.
  This subset states that, during execution, all boxes reachable from available variables are alive.
  The combination of ownership and borrow checking derives the full *No dangling references* property, and the details are described in the [soundness section](./soundness.md#satisfying-well-formedness-properties).

TODO: USE THIS OR THROW IT AWAY

The following subsections presents an implementation-level definition of the formal concepts described in the [memory management section](./memory-management.md).

### Definitions

***Decompose a linear value***
Given an instance \( l \) of a linear class \( L \) with fields of types \( (T_1, T_2, \ldots, T_n) \), also called a linear value \( l: L \), decomposing the value using \( decomp(l) \) yields the list of field values of \( L \), namely \( (l_1: T_1, l_2: T_2, l_3: T_3, \ldots, l_n: T_n) \).

***Reachability:***
During program execution, a value \(v\), which can be an instance of any class, can reach another value \(v'\) if there exists a sequence of operations \(Op_1, Op_2, Op_3, \ldots, Op_n\) such that each \(Op_i\) is one of the following operations and \(Op_1 \circ Op_2 \circ Op_3 \circ \ldots \circ Op_n (v) = v'\):

- If the input is a box \(b: \text{Box}[T, \ldots]\), the operation returns \(deref(b)\).
- If the input is a linear value \(l: L\), the operation returns one of the elements of \(decomp(l)\).

```Scala
TODO: AN EXAMPLE OF REACHABILITY
```

***Direct box:***
Given a value \(v\), the list of direct boxes of \(v\), denoted \(direct(v)\), consists of all boxes \(b\) such that \(deref(b) = v\).

***Available variable:***
A variable is available in the program scope at a point during execution if referring to the variable in an expression does not at that point produce a compiler error.
In the imem context, this means:

- The variable is defined in one of the enclosing scopes.
- If the variable has a linear type, it has not expired.
- If the declared type of the variable includes type parameters instantiated with capture sets, any linear capability in those capture sets has not expired.

***Resource***:
A resource is a value \(v\) that is reachable from available variables and satisfies the condition \(|direct(v)| \geq 1\).

***Aliveness:***
At each point during program execution, a box \(b: \text{Box}[T, \ldots]\) is alive if its type does not include any type parameter instantiated with a capture set that contains an expired linear capability.

TODO: ADD THIS


***dangling-box-unavailability-lemma:***
Let \(v: V\) be a value that reaches a box instance \(b': \text{Box}[T', Owner']\). The value \(v\) can be either a box \(b: \text{Box}[T, Owner]\) or a linear value \(l: L\). If a lifetime capability \(lf\) is in \(Owner'\), \(lf \in Owner'\), then \(lf\) occurs in at least one capture-set instantiation of a type parameter of \(V\).

If a box \(b\) reaches another box \(b'\), then either \(b = b'\), or \(deref(b)\) reaches \(b'\).
If a linear value \(l\) reaches a box \(b'\), then at least one field value \(f: F \in decomp(l)\) reaches \(b'\).

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

- If \(v\) is a linear value \(l: L\), then some field value \(f: F \in decomp(l)\) reaches \(b'\).
  By the induction hypothesis there exists a type parameter \(p\) of \(F\) that is instantiated with a capture set containing \(lf\).
  Each type parameter of \(F\) is instantiated either with types defined in \(L\) or with types built from type parameters of \(L\).
  Since linear classes do not allow nested type definitions, there exists a type parameter of \(L\) that \(p\) is instantiated with, and this instantiation contains a capture set that includes \(lf\).

If a box instance \(b': \text{Box}[T', Owner']\) is not alive, then some lifetime capability \(lf \in Owner'\) has expired.
Suppose another box instance or a linear value of type \(V\) reaches \(b'\).
By the *dangling-box-unavailability-lemma*, the capability \(lf\) appears in a capture-set instantiation of a type parameter of \(V\).
Consequently, any variable that refers to this box instance or linear value has a type that includes an expired lifetime capability and is therefore unavailable.
It follows that every box reachable from an available variable must be alive, which establishes that the *No dangling boxes* property always holds.

TODO: ADD THIS

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
The value \(t\) is not a resource because \(direct(t) = 0\).
Moreover, the only box that provides access to \(t\) is \(b\), and \(b\) is inaccessible inside `moveAction`.
Also, because the `newBox` function requires a resource argument with an empty capture set, the program, inside `moveAction`, cannot create a new box that turns \(t\) into a resource again.

After `moveAction`, since \(t\) was an escape-checked argument in `moveAction`; the program can only access \(t\) after `moveAction` if `moveAction` returns \(t\).
In that case, \(t\) is still not a resource, and the statement of the lemma continues to hold.

------------------------------------------------------------------------------

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

##### Resource

The `Box` class constructor is private, which means that a program can create a box only through the imem interfaces.
These interfaces take a resource as input and return a box that points to that resource.
imem assumes that the only Scala reference to the resource is the one passed to the box constructor.
In other words, the program does not store the resource’s Scala reference, or any object that can reach that resource, elsewhere in memory.
In the following example, the program creates two boxes that refer to the same resource:

```Scala
TODO: CREATE TWO BOXES WITH TWO LINEAR RESOURCE THAT HAS A FIELD POINTING TO THE SAME NONLINEAR OBJECT
```

imem controls only access to a box’s resource, not the resource reaching memory.
Therefore, if a program creates multiple resources that point to the same underlying object via Scala references, imem regulates access to those resources, not to the object they point to.
Managing access to the shared object itself remains the program's responsibility.

It is important to note that imem’s static guarantees no longer apply when a program reaches the non-linear part of memory.
A box’s resource must be a linear class, and access to linear instances is statically controlled according to Scinear rules.
If the resource's class’s fields are also linear, Scinear statically manages access to the memory associated with those fields as well, and this property propagates transitively.
For this reason, imem recommends keeping memory linear as much as possible.

One important guideline for linear memory is to replace direct linear fields with box fields that points to linear classes.
For example, in the following `LinearBinTree` class, the fields have type `LinearBinTree`:

```Scala
TODO: A LINEAR_BIN_TREE CLASS THAT HAS TWO FIELDS LEFT AND RIGHT BOTH ARE LINEAR_BIN_TREE.
```

According to imem’s guarantees, this implementation is correct.
However, when the program has access to a `LinearBinTree` instance's children, it cannot borrow them immutably or mutably.
It also cannot reference both of a node's children simultaneously or iterate over the tree without consuming the tree and destructing and reconstructing it during the traversal.

```Scala
TODO: A BOX THAT POINTS TO A TREE WITH THREE NODES AND ACCESSING THE LEAF IS NOT POSSIBLE, JUST HAVING A REFERENCE TO THE ROOT IS POSSIBLE
```

The following example shows that when a `LinearBinTree` class refers to its children through `Box` instances, it is possible to hold a mutable reference to one child and an immutable reference to the other at the same time.

```Scala
TODO: LinearBinTree WITH BOXES AS FIELDS AND TWO REFERENCES, AN IMMUTABLE REFERENCE TO LEFT CHILD AND MUTABLE REFERENCE TO RIGHT CHILD.
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
  This option enables the program to get back the borrowing's source.
  In this case, the program can unlock the value holder only by using an instance of the associated lifetime key, which causes the lifetime to expire.

- **`NeverUsableKey` with any lifetime set:**
  This option is useful when the program no longer needs the source of the borrow and only requires the borrowed reference for use.
  Because `NeverUsableKey` is an opaque type, the program cannot create instances of it, and therefore cannot unlock the value holder.

There is no static enforcement that restricts valid programs to these two cases.
This means that a program can misuse these parameters and break imem’s guarantees:

```Scala
TODO: A BORROWING THAT SET THE KEY AS OBJECT AND USE AN OBJECT INSTANCE TO OPEN THE HOLDER.
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

imem assumes that programs do not alter context's accumulated owners.
Due to heavy limitations on type capturing the universal capability, `cap`, I was unable to find a concrete example in which this behavior breaks imem’s rules without also causing compile-time errors.
