# Borrow Checking

This section describes how imem manages borrowing and the references that result from it.

## Borrow Checking Goals

### Definitions

***Dereferencing a reference:***
Given a reference $r: \text{Mut}[T, \ldots]$ or $r: \text{Immut}[T, \ldots]$ whose resource is $t: T$, dereferencing the reference, $deref(r)$ would result in $t$.
That is, $deref(r) = t$.

### Goals

***Box outliving references invariant:***  
During execution, for any box \(b : \text{Box}[T, O]\) and any immutable reference \(r : \text{ImmutRef}[T, O']\) or mutable reference \(r : \text{MutRef}[T, O']\), if \(deref(b) = deref(r)\), then \(b\) outlives \(r\).
That is, \(O \subset O'\).

***Stacked borrows:***  
At any point during program execution, consider a resource \(t : T\) with the following:

- A box \(b : \text{Box}[T, \ldots]\) such that \(deref(b) = t\).
- A sequence of mutable references \(m_1, m_2, \ldots, m_N\), with \(N \ge 0\), such that \(deref(m_j) = t\) and, for \(1 \le j < N\), \(m_j\) is borrowed before \(m_{j+1}\).
- A sequence of immutable references \(i_1, i_2, \ldots, i_M\), with \(M \ge 0\), such that \(deref(i_j) = t\).

The following always hold:

- If the program uses an available variable that refers to \(b\), then all mutable references \(m_1, \ldots, m_N\) and all immutable references \(i_1, i_2, \ldots, i_M\) become unavailable.
- If the program uses an available variable that refers to \(m_j\), then all mutable references \(m_{j+1}, \ldots, m_N\) and all immutable references \(i_1, i_2, \ldots, i_M\) become unavailable.

***Immutable reference constant view:***  
Any box reachable from an immutable reference remains constant for as long as the immutable reference lives.

imem achieves *Box outliving references invariant* using [Reference Ownership Management](#reference-ownership-management), *Stacked borrows* using [Box and Reference Holding](#box-and-reference-holding) and (#reference-owner-aggregation), and *Immutable reference constant view* using [Static Permission Management](#static-permission-management-of-operations).

## Reference Ownership Management

References, similar to boxes, are owned by a set of lifetime capabilities.

```Scala
class ImmutRef[T, Owner^](
  private[imem] val tag: InternalRef[T]#Tag,
  private[imem] val internalRef: InternalRef[T]
)

class MutRef[T, Owner^](
  private[imem] val tag: InternalRef[T]#Tag,
  private[imem] val internalRef: InternalRef[T]
) extends scinear.Linear
```

Both the `ImmutRef` and `MutRef` classes include an `Owner^` capture-set type parameter, which the program instantiates with a set of lifetimes.

imem does not provide interfaces for creating references out of thin air; instead, references are borrowed from boxes or from other references.

### Borrowing a Box

To borrow a box, imem provides the following interfaces:

```Scala
def borrowImmutBox[@scinear.HideLinearity T, Owner^, ..., newOwner^ >: {Owner}, ...](
  self: Box[T, Owner]^
)(...): (ImmutRef[T, newOwner], ...) = ...

def borrowMutBox[@scinear.HideLinearity T, Owner^, ..., newOwner^ >: {Owner}, ...](
  self: Box[T, Owner]^
)(...): (MutRef[T, newOwner], ...) = ...
```

`borrowImmutBox` and `borrowMutBox` borrow a box immutably and mutably, respectively.
Both operations take a `newOwner` type parameter and return a reference whose owner is `newOwner`.
Note that the resulting reference does not capture `self`. Therefore, escape-checking `self` has no effect on the returned reference.

For both functions, the owner type parameter `newOwner` of the returned reference must be a superset of the `Owner` type parameter.
As a result, the reference's owners include all the lifetime capabilities owners of the box from which the reference is borrowed.
By enforcing the constraint `newOwner^ >: {Owner}`, imem ensures the *Box outliving references invariant* when borrowing a box.

### Borrowing an Immutable Reference

At any point, multiple immutable references may simultaneously share a resource.
To enable this, the program first borrows a box immutably and then reborrows the resulting immutable reference as many times as needed.
The following is imem's interface for reborrowing an immutable reference:

```Scala
def borrowImmut[@scinear.HideLinearity T, Owner^, ..., newOwner^ >: {Owner}, ...](
  self: ImmutRef[T, Owner]^
)(...): ImmutRef[T, newOwner] = ...
```

`borrowImmut` for an immutable reference is similar to `borrowImmutBox`.
The returned reference does not capture `self`.
Therefore, even if `self` is escape-checked, the program can copy the returned reference and store it freely.
In addition, the `newOwner` type parameter, with the constraint `newOwner^ >: {Owner}`, ensures that reborrowed immutable references live no longer than the original immutable reference, in line with the *Box outliving references invariant*.

### Borrowing a Mutable Reference

imem allows the program to reborrow a mutable reference either mutably or immutably.

#### Mutably borrowing a mutable reference
`borrowMut` mutably borrows a mutable reference and produces another mutable reference.

```Scala
def borrowMut[@scinear.HideLinearity T, Owner^, ..., newOwnerKey, newOwner^ >: {Owner}, ...](
  self: MutRef[T, Owner]^
)(...): (MutRef[T, newOwner], ...) = ...
```

The signature is identical to that of `borrowMutBox`, and ownership monotonicity is preserved.
This means that the owner of the derived reference, `newOwner`, is a superset of the owner of the original reference, `Owner`.
As a result, the original reference, and transitively the box, outlives the derived reference.

### Immutably borrowing a mutable reference

To immutably borrow a mutable reference, imem provides `borrowImmut`:

```Scala
def borrowImmut[@scinear.HideLinearity T, Owner^, ..., newOwner^ >: {Owner}, ...](
  self: MutRef[T, Owner]^
)(...): (ImmutRef[T, newOwner], ...)
```

The same as `borrowMut`, except that `borrowImmut` returns an immutable reference.

## Box and Reference Holding

### Value Holder

To ensure that `MutRef` and `Box`, or `ImmutRefs`, do not coexist for the same resource at the same time and to follow the [Stacked Borrows Model](../background/stacked-borrows.md), imem uses a mechanism that ties the existence of one entity to the non-existence of another.
In the [memory management section](../imem/memory-management.md), \(hold\) references are responsible for tying reference accesses to the expiration of other references.
In the implementation, a combination of `ValueHolder` and `Lifetime` enables this mechanism.

```Scala
class ValueHolder[KeyType, T](private[imem] val value: T) extends scinear.Linear
```

`ValueHolder` is a simple linear class parameterized by `KeyType` and containing a field of type `T` that stores a value of any type.
The value can only be accessed through the following function:

```Scala
def unlockHolder[KeyType, @scinear.HideLinearity T](
  key: KeyType,
  holder: ValueHolder[KeyType, T]^
): T^{holder} = holder.value
```

The `unlockHolder` function returns the value only when a key instance that matches the `KeyType` type parameter of the holder is provided.
To make the function usable by other imem interfaces, the holder captures the universal capability, and the returned value captures the holder in case the holder is subject to [escape checking](../background/capturing-types.md#escape-checking).

Imem ties a `ValueHolder` to a `Lifetime` by using an internal opaque dependent type as the key.  
In this way, unlocking the holder always causes the associated lifetime to expire.

```Scala
class Lifetime[OwnersInput^] extends scinear.Linear, caps.Capability:
  type Owners^ = {OwnersInput, this}
  opaque type Key = Object

  def getKey[O^](): Key = Object()
end Lifetime
```

In the Lifetime class, `Key` is an opaque type member.
The only way to obtain an instance of this type is through the `getKey` method of `Lifetime` because the type is opaque outside the class.  
At the same time, because `Key` is a type member, each instance of type `Lifetime` has its own [path-dependent](../background/dependent-types.md) `Key` type that is unique to that instance.

As shown below, a `ValueHolder[lf.Key, ...]` can only be opened by `lf` and by no other lifetime.

```Scala
TODO: A EXAMPLE OF VALUEHOLDER[LF.KEY, ...]
```

### Borrowing and Holder Results

The `Box` and `MutRef` references are linear types.
This means that when the program borrows them, using the borrowing interfaces, the instance passed to the interfaces expires.
But the program might need the passed references and imem provides the expired instances in `ValueHolder` tied to the `Lifetime` capability that the newly borrowed reference is associated with.

The following list presents all borrowing functions provided by imem:

```Scala
def borrowImmutBox[@scinear.HideLinearity T, Owner^, ..., newOwnerKey, newOwner^ >: {..., Owner}, ...](
  self: Box[T, Owner]^
)(...): (ImmutRef[T, newOwner], ValueHolder[newOwnerKey, Box[T, Owner]^{self}]) = ...

def borrowMutBox[@scinear.HideLinearity T, Owner^, ..., newOwnerKey, newOwner^ >: {..., Owner}, ...
  self: Box[T, Owner]^
)(...): (MutRef[T, newOwner], ValueHolder[newOwnerKey, Box[T, Owner]^{self}]) = ...

def borrowMut[@scinear.HideLinearity T, Owner^, ..., newOwnerKey, newOwner^ >: {.., Owner}, ...
  self: MutRef[T, Owner]^
)(...): (MutRef[T, newOwner], ValueHolder[newOwnerKey, MutRef[T, Owner]^{self}]) = ...

def borrowImmut[@scinear.HideLinearity T, Owner^, ..., newOwnerKey, newOwner^ >: {..., Owner}, ...
  self: MutRef[T, Owner]^
)(...): (ImmutRef[T, newOwner], ValueHolder[newOwnerKey, MutRef[T, Owner]^{self}])
```

All borrowing functions return a pair.
The first element of the pair is the borrowed reference, and the second element is a value holder locked with the `newOwnerKey` type parameter and storing a value of the same type as the `self` parameter.  
The `T` type parameter of the value holder is instantiated with the type of `self`, which captures `self` in case `self` is subject to [escape checking](../background/capturing-types.md#escape-checking).

The following example demonstrates how a program can borrow a reference from another reference and then retrieve the original reference:

```Scala
TODO: AN EXAMPLE THAT STARTS WITH DEFINING A LIFETIME, THE BORROWING A BOX, AND RETRIEVING IT. THE SAME FOR A MUTABLE REFERENCE.
```

The program borrows a box or a reference using `lf.Owners` as the owners of the borrowed reference and `lf.Key` as the locking key for the holder.
It then unlocks the holder by calling `lf.getKey`, which retrieves the original box or reference, causes the lifetime to expire, and consequently makes the borrowed reference unavailable.

## Static Permission Management of Operations

### Using References

The main purpose of references is to provide access to a resource after it has been borrowed by the program.
An immutable reference provides only read access through the `read` function, and a mutable reference provides both read and write access through the `write` function.

```Scala
def read[@scinear.HideLinearity T, Owner^, @scinear.HideLinearity S, ...](
  self: ImmutRef[T, Owner]^,
  readAction: ... ?-> T^ ->{...} S
)(...): S =

def write[@scinear.HideLinearity T, Owner^, @scinear.HideLinearity S, ...](
  self: MutRef[T, Owner]^,
  writeAction: ... ?->{...} T^ ->{...} S
)(...): S = ...
```

The `read` function takes an immutable reference, `self`, together with a continuation-passing-style argument, `readAction`.
Within `readAction`, the program is granted access to the underlying resource.
The signature of `readAction` is `... ?-> T^ ->{...} S`, where the `T^` parameter is the means by which the action accesses the resource.
This access is escape-checked because the `T^` parameter captures the universal capability.
At the same time, the return type `S` does not capture the argument, which prevents the action from leaking or storing the resource beyond its scope.
The `read` function returns the result produced by the action.

```Scala
TODO: AN EXAMPLE WHER AN IMMUTABLE REFERENCE BORROWED AND THEN IS USED BY READ FUNCTION TO PRINT THE Int RESOURCE INSIDE IT.
```

The `write` function follows the same structure as the `read` function.
The key difference is that the `self` parameter of `write` is a mutable reference, `MutRef`.
Because `MutRef` is linear, the program cannot access the mutable reference after calling the `write` function, nor within the body of `writeAction`.
As a result, when the program accesses the resource of a mutable reference through the `write` function, the mutable reference is consumed and cannot be used again.

### Operation Rules

The `read`, `write`, and `derefForMoving` functions each take a continuation-passing-style argument that provides controlled access to the resource of an immutable reference, a mutable reference, and a box, respectively.
Within the scope of these actions, the program must not have access to all interfaces provided by imem.

More precisely, the following restrictions apply.

***Inside `write`’s `writeAction`:***
The program must not be able to call `moveBox` or `derefForMoving`.
Allowing these operations could produce references or boxes that point to a moved box, which violates the *One direct box* goal.

```Scala
TODO: AN EXAMPLE WHEN INSIDE A WROTE A BOX IS MOVED AND THEN THE MOVED BOX HAVE TWO BOXES POINTING TO IT.
```

***Inside `read`’s `readAction`:***
As with `writeAction`, the program must not be able to call `moveBox` or `derefForMoving`.
In addition, the program must not be able to call `setBox` or `swapBox`, because these operations would modify a box while it is reachable through an immutable reference, which violates the *Immutable reference constant view* goal.
For the same reason, the program must not be able to call `borrowMutBox`, `borrowMut`, or `write`, since these operations could create a mutable reference to a resource that is already reachable through an immutable reference.

```Scala
TODO: AN EXAMPLE OF MODYFING A BOX INSIDE READ SCOPE CAUSING PROBLEM. ALSO MUTABLY BORROW A BOX INSIDE READ AND RETURNING IT CAUSING IMMUTABLE REFERNCE REACHING MUTABLE difference
```

The following table summarizes the scopes and the actions permitted within them:

| Access type | Scopes that have it | Enables |
|---|---|---|
| read | default scope, `derefForMoving`, `writeAction`, `readAction` | `borrowImmutBox`, `borrowImmutBox`, `read` |
| write | default scope, `derefForMoving`, `writeAction` | `setBox`, `swapBox`, `borrowMutBox`, `borrowMut`, `write` |
| move | default scope, `derefForMoving` | `moveBox`, `derefForMoving` |

### Write and Move capabilities

## Reference Owner Aggregation

### Why
### Context


