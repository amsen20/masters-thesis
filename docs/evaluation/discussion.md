# Discussion

This section compares the performance of imem with other baselines in the case study of implementing a statically memory-managed safe linked list.

## Functionality

| Implementation | push & pop | peek | peekMut | Consuming Iterator | Mutable Iterator | Static Memory Management | Static Mutability Control |
|---|---|---|---|---|---|---|---|
| Rust | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ |
| Vanilla Scala | $\checkmark$ | $\checkmark$ | $\checkmark$* | $\checkmark$ | $\checkmark$* | $\times$ | $\times$ |
| Linear Scala | $\checkmark$ | $\times$ | $\times$ | $\checkmark$ | $\times$ | $\checkmark$ | $\times$ |
| imem | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ | $\checkmark$ |

Rust enforces ownership and borrow checking, resulting in static memory management and static mutability control.
As a result, the linked list can implement all required functionalities, including `push`, `pop`, `peek`, `peekMut`, a consuming iterator, and a mutable iterator.

Vanilla Scala uses garbage collection for references and does not provide static memory management.
Also, vanilla Scala has no definition of mutability through a reference.
This means that enabling `peekMut` and a mutable iterator requires returning a `Node` that exposes the internal structure, which the implementation should avoid if possible.

Linear Scala enforces linearity, through the Scinear plugin, and linearity enables statically managing memory.
However, linear memory does not support mutability.
Therefore, mutable features such as `peekMut` and a mutable iterator are not possible, and static mutability control is also not supported.
In addition, immutable peeking is not possible.  
This limitation exists because pure linearity does not support borrowing.
In a purely linear memory model, when the program gains access to a part of the memory, that memory becomes inaccessible from any other location.
Otherwise, such access would violate the linear memory tree structure.

imem combines linearity from Scinear with capture checking and dependent types that are already part of the Scala type system.
Together, these features enable borrow checking and ownership tracking.
As a result, similar to Rust, imem supports full functionality, including static memory management and static mutability control.

## Verbosity

Both the Rust and the vanilla Scala implementations are short and concise, whereas the Linear Scala implementation is slightly longer, and the imem implementation is significantly longer.
Linear Scala must follow a pattern in which the program decomposes a linear value into its fields and then reconstructs it so that no field is lost when accessing others.
As a result, the code contains many `match` and `case` patterns.
A similar linear access pattern appears in the imem code because resources are also linear values.

The baseline that is able to implement all the features alongside imem is Rust, and the following are the main differences between imem and Rust that make an imem program longer:

- Rust's `take()`, `as_ref()`, and `as_mut()` methods: These are all methods returning `Option<T>`, which makes dealing with Rust's borrow checking easier.
  In contrast, in imem, the program has to use borrowing interfaces to borrow the box pointing to the `Option[Box[T]]` and the box that the option is pointing to, and use `read`, `write` to access their resources.
- Borrowing non-boxes: In Rust, the `head` field in `List` and the `elem` and `next` fields in `Node` are not `Box`es, but the program is able to have mutable and immutable references to them that are borrow checked.
  On the hand, imem is only able to have mutable and immutable references to box's resource, as a result the program has to wrap everything in boxes and access them through layer by layer `read`/`write` functions.
- Language support for borrowing and dereferencing: As an example, `&node.elem` expression in Rust, first dereferences `node`, accesses the `elem` field then borrows the reference.
  Because imem does not have any language support, the program have to use the contuniation-passing-style interfaces to dereference the `node`.

The [future works](../conclusion/future-works.md) explains how small improvements can fill this gaps.

Both Rust and vanilla Scala implementations are short and concise, whereas the Linear Scala implementation is slightly longer, and the imem implementation is significantly longer.
Linear Scala follows a pattern in which a linear value is decomposed into its fields and then reconstructed so that no field is lost when accessing others.  
As a result, the code contains many `match` and `case` patterns.
A similar linear access pattern appears in the imem code because resources are also linear values.

Rust is the only baseline that supports all features alongside imem.
The following are some of the main differences between imem and Rust that make an imem program longer:

- **Rust's `take()`, `as_ref()`, and `as_mut()` methods:** These methods belong to `Option<T>` and simplify interaction with Rust’s borrow checker for this enum.
  In contrast, imem requires the use of borrowing interfaces to borrow the box that points to `Option[Box[T]]`, and then the box that the option points to.
  The program must also use `read` and `write` operations to access the underlying resources.
- **Borrowing non-box values:** In Rust, the `head` field in `List`, as well as the `elem` and `next` fields in `Node`, are not wrapped in `Box<T>`.
  However, the program is able to have borrow-checked mutable and immutable references to these fields.
  In contrast, imem allows mutable and immutable references only to the resources inside boxes.
  As a result, the program has to wrap all values must in boxes and access them layer by layer using `read` and `write` functions.
- **Language support for borrowing and dereferencing:** For example, the expression `&node.elem` in Rust first dereferences `node`, then accesses the `elem` field, and finally creates a reference to it.
  Since imem does not provide built-in language support for such operations, dereferencing must happen through using continuation-passing-style interfaces.

The [future works](../conclusion/future-works.md) section explains how small improvements can fill these gaps.


## Error Clarity

The [imem in practice](./imem-in-practice.md#a-familiar-example) section examines the implementations of the `longest` function in both Rust and imem.
The incorrect Rust program produces the following complete error message:

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

For comparison, the following is the same mistake's error message in Scala using imem:

```
[error]    | ...
[error]    | I tried to show that
[error]    |   imem.ImmutRef[String, O1] | imem.ImmutRef[String, O2]
[error]    | conforms to
[error]    |   imem.ImmutRef[String, O3]
[error]    | but none of the attempts shown below succeeded:
[error]    |
[error]    |   ==> imem.ImmutRef[String, O1] | imem.ImmutRef[String, O2]  <:  imem.ImmutRef[String, O3]
[error]    |     ==> imem.ImmutRef[String, O1]  <:  imem.ImmutRef[String, O3]
[error]    |       ==> O3  <:  O1
[error]    |         ==> subcaptures {O3} <:< {O1} in open varState
[error]    |           ==> {O1} accountsFor O3  = false
[error]    |           ==> {O1} accountsFor cap  = false
[error]    |
```

The imem version is less understandable and more difficult to debug.
In fact, the imem error is not a borrow-checking error.
Instead, it explains the type checker’s attempts to satisfy the capture-checking rules under the given constraints.
Currently, I am not aware of a proper solution to address this problem without adding language support for imem.
