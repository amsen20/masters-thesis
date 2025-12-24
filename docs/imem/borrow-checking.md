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

## Value Holder

## Reference Owner Aggregation

## Static Permission Management of Operations

### Using References