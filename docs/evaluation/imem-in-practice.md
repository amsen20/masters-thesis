# imem in practice

This section explains how not following imem rules results in compile errors. This section also demonstrates the practicality of imem.

## Linked List Implementation

The linked list implementation, described in the [imem section](./imem.md), follows the imem static rules.
This subsection examines two selected parts of the implementation and demonstrates the errors that would result from modifying them.

### Mutable Iterator Reference Owner Set

In the `iterMut` function, the type parameter `O2^` is statically bounded as a superset of the `O1^` and `O3^` type parameters, `O2 >: {O1, O3}`.
In this function, `O2` denotes the iterator and its internal reference owner set, `O1` denotes the list owner set, and `O3` denotes the context’s aggregated owner set.
The following shows the `iterMut` function signature:

```Scala
def iterMut[T <: scinear.Linear, @caps.use O1^, O3^, O2^ >: {O1, O3}, @caps.use WC^, MC^](
  self: MutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^{O3}
): MutableIterator[T, O1, O2] = ...
```

The `self` argument is a mutable reference with `O2` as its owner set and `O1` as the owner set of the reference's resource, `List[T, O1]`.
In addition, `O3` is the capture set of the capturing type `Context[WC, MC]^{O3}`, which represents the context.

The intention behind this relation is that the mutable iterator needs to borrow fresh mutable references from the list throughout the iteration.
Moreover, the type of the internal reference, `MutableIterator.boxToLink`, namely `Box[MutRef[Link[T, O1], O2], O2]`, remains unchanged during iteration.
This property means that every freshly borrowed reference must have the same `O2` owner set.

Based on imem's [borrow checking rules](../imem/borrow-checking.md), in each borrow, the resulting reference owner set must be a superset of the owner set of the borrowed box or reference and the owners aggregated by the context, which results in the invariant `O2^ >: {O1, O3}`.
But, what happens if the `iterMut` function does not explicitly state this type bound?
The following shows the function signature without this bound:

```Scala
def iterMut[T <: scinear.Linear, @caps.use O1^, O3^, O2^, @caps.use WC^, MC^](
  self: MutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^{O3}
): MutableIterator[T, O1, O2] = ...
```

And the following is the resulting compilation error:

```
[error] 672 |      val (headRef, listHeadHolder) = borrowMutBox[Link[T, O1], O1, {ctx}, NeverUsableKey, {O2}, {WC}, {MC}](list.head)
[error]     |                                                                                            ^
[error]     |Type argument scala.caps.CapSet^{O2} does not conform to lower bound scala.caps.CapSet^{ctx, O1}
[error]     |---------------------------------------------------------------------------
[error]     | Explanation (enabled by `-explain`)
[error]     |- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
[error]     | ...
[error]     |     ==> subcaptures {ctx, O1} <:< {O2} in open varState
[error]     | ...
[error]     | The tests were made under the empty constraint
```

This is a capture-checking trace, which is typically a long list of unsuccessful attempts of the compiler to match inferred and annotated capture sets, that highlights the first borrow that depends on the invariant `O2^ >: {O1, O3}`.
This borrow occurs during the creation of the mutable iterator and reports that `subcaptures {ctx, O1} <:< {O2}` in `open varState` does not hold, as `borrowMutBox` signature requires it.

### Write and Move Capability Annotations

All the functions in the linked list implementation using imem get `WC^` and `MC^` type parameters to instantiate the context instance (`Context[WC, MC]`) type parameters they are getting.
`WC^` and `MC^` are type parameters instantiated by the write and move capability, respectively.

All the functions that include operations that require write access or move access annotate the `WC^` or `MC^` type parameter using `@caps.use` annotation.
The following is an example of this annotation in `pop` function:

```Scala
def pop[..., @caps.use WC^, @caps.use MC^](...)(
  using ctx: Context[WC, MC]^
): ... = ...
```

The reason for this behavior is that the `pop` function both updates `list.head` and then moves the popped element.
If the `pop` function did not annotate `WC^` and `MC^` with `@cap.use`, as shown below:

```Scala
def pop[..., WC^, MC^](...)(
  using ctx: Context[WC, MC]^
): ... = ...
```
then the following would be the compilation error:

```
[error]     | ...
[error] 193 |      val (listRef, selfHolder) = borrowMut[List[T, O1], O2, {ctx, O2}, lf.Key, lf.Owners, {WC}, {MC}](self2)
[error]     |                                                                                            ^^
[error]     |  Capture set parameter WC leaks into capture scope of method pop.
[error]     |  To allow this, the type WC should be declared with a @use annotation
[error]     | ...
[error] 267 |    derefForMoving[Link[T, O1], O3, {ctx}, Option[Box[T, O3]], {WC}, {MC}]
[error]     |                                                                      ^^
[error]     |  Capture set parameter MC leaks into capture scope of method pop.
[error]     |  To allow this, the type MC should be declared with a @use annotation
[error]     | ...
```

These errors occur at every point where the function uses imem interfaces or other functions that annotate `WC` or `MC` with `@cap.use`.
Because the function does not explicitly state that it captures `WC` or `MC`, the error informs the programmer that `WC` and `MC` are leaking into the function’s capture set.

## A Familiar Example

The [Rust documentation book](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html), in Chapter 10, Section 3, explains how to correctly annotate the `longest` function.  
This function takes two string references and returns a reference to the longer string:

```Rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

Implementing the function as above would result in the compile error below:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

Now let's implement the same function using imem.
The following implements the `longest` function using the imem library:

```Scala
def longest[O4^, O1^ >: {O4}, O2^ >: {O4}, O3^ >: {O1, O2}, WC^, MC^](
  a: ImmutRef[String, O1],
  b: ImmutRef[String, O2]
)(using ctx: Context[WC, MC]^{O4}): ImmutRef[String, O3] =
  val isALonger = read[String, O1, Boolean, O4, WC, MC](a, aVal =>
    read[String, O2, Boolean, O1, WC, MC](b, bVal =>
      aVal.length > bVal.length
    )
  )
  if isALonger then
    borrowImmut[String, O1, O4, NeverUsableKey, O3, WC, MC](a)
  else
    borrowImmut[String, O2, O4, NeverUsableKey, O3, WC, MC](b)
```

Similar to the Rust version, in which all references are annotated with the `'a` lifetime to inform the borrow checker that both arguments must outlive the result, this version uses `O3^ >: {O1, O2}` to inform the type checker that the returned reference expires earlier than both the `a` and `b` references.
If the function does not explicitly state that `O3^` is a superset of `O1` or `O2`, one of the following errors occurs, respectively:

```
[error] 17 |    borrowImmut[String, O1, O4, NeverUsableKey, O3, WC, MC](a)
[error]    |                                                ^
[error]    | Type argument O3 does not conform to lower bound scala.caps.CapSet^{O1}
[error]    | ...
[error] 22 |    borrowImmut[String, O2, O4, NeverUsableKey, O3, WC, MC](b)
[error]    |                                                ^
[error]    | Type argument O3 does not conform to lower bound scala.caps.CapSet^{O2}
```

## Using Iterators

The following examples show how imem enforces that the program has access to either the list or the mutable iterator, but not both at the same time, and that the list is no longer accessible after a consuming iterator is created.

```Scala
// create an empty list with {ctx} as its lifetime
val list = newBoxFromBackground(newLinkedListFromBackground[LinearInt, {WC}, {MC}])

// a lifetime for the mutable iterator
val lf1 = Lifetime[{ctx}]()

// borrow the list mutably to use it for the mutable iterator
val (listRef, listHolder) = imem.borrowMutBox[List[LinearInt, {ctx}], {ctx}, {ctx}, lf1.Key, lf1.Owners, {WC}, {MC}](list)
// create the mutable iterator
val mutIter = iterMut[LinearInt, {ctx}, lf1.Owners, lf1.Owners, {WC}, {MC}](listRef)
// don't need `iter` anymore, so consume it
mutIter.consume()

// get the list back
val list2 = unlockHolder(lf1.getKey(), listHolder)

// create a consuming iterator from the list
val consumingIter = intoIter[LinearInt, {ctx}, {ctx}, {WC}, {MC}](list2)
// wrap it in a box
val iterBox = newBox[ConsumingIterator[LinearInt, {ctx}], {ctx}](consumingIter)
// don't need `iterBox` anymore, so consume it
iterBox.consume()
```

The program begins by creating the list using the `newLinkedListFromBackground` and `newBoxFromBackground` functions, which create a `List` and then a `Box` for the list, respectively, with `{ctx}` as their lifetime set.
Here, `{ctx}` plays a role similar to the `'static` lifetime in Rust and serves as the base lifetime that every newly borrowed references' owner set must be a superset of.
The program then mutably borrows the list to create a mutable reference.
Because `list` is a linear variable, it is no longer accessible after the borrow.
The program below attempts to use the list after borrowing it:

```Scala
...
// borrow the list mutably to use it for the mutable iterator
val (listRef, listHolder) = imem.borrowMutBox[List[LinearInt, {ctx}], {ctx}, {ctx}, lf1.Key, lf1.Owners, {WC}, {MC}](list)
list
...
```

The program above triggers the following compile error:

```
...
[error] 36 |  list
[error]    |  ^^^^
[error]    |  Linear value list is being used twice, or is not accessed directly.
...
```

This example shows that the program can regain access to the list only by unlocking `listHolder`.
To unlock the value holder, the program must use `lf1`’s key, which causes `lf1` and `mutIter` to expire.
As a demonstration, the following program moves the use of `mutIter` to after unlocking `listHolder`:

```Scala
...
// get the list back
val list2 = unlockHolder(lf1.getKey(), listHolder)
// don't need `iter` anymore, so consume it
mutIter.consume()
...
```

The above code results in the compilation error below:

```
...
[error] 43 |  mutIter.consume()
[error]    |  ^^^^^^^
[error]    |Linear value lf1 is mentioned in the type of mutIter but is either not in scope or used already.
...
```

A similar argument applies to `list2` and `consumingIter`.
Because `list2` is linear, the program can no longer access `list2` after creating `consumingIter`.

## More Details Examples

The [should work](https://github.com/amsen20/imem/blob/main/src/test/scala/ShouldWorkSuite.scala) and [should not work](https://github.com/amsen20/imem/blob/main/src/test/scala/ShouldNotWorkSuite.scala) tests in the imem implementation provide more detailed examples of using all imem's functions, as well as the linked list implementation built on imem.
The `ShouldWorkSuite` contains use cases of imem, the linked list, and their expected behavior.
The `ShouldNotWorkSuite` contains functions that intentionally produce compilation errors, which illustrate violations of imem’s rules.
