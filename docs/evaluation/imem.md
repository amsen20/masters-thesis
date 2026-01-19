# imem

This section demonstrates how a linked list implementation appears when using the imem library.

The imem library uses capture checking in different parts of the library, and capture checking is currently an experimental compiler feature that is still under development.
As a result, the compiler has limited ability to infer capture set type parameters, and the inference also contains [bugs](TODO) that are inconsistent with the capture checking rules.
For this reason, this implementation states all type parameters in explicitly.
This explicit mentioning reduces the expressiveness of the implementation.

## Internal Structures

The internal structures of the linked list are implemented in imem as follows:

```Scala
class List[T <: scinear.Linear, O^](
  _head: Box[Link[T, O], O] = newBox[Link[T, O], O](None)
) extends scinear.Linear:
  val head: Box[Link[T, O], O]^{this} = _head
end List

type Link[T <: scinear.Linear, O^] = Option[Box[Node[T, O], O]]

class Node[T <: scinear.Linear, O^](_elem: Box[T, O], _next: Box[Link[T, O], O]) extends scinear.Linear:
  val elem: Box[T, O]^{this} = _elem
  val next: Box[Link[T, O], O]^{this} = _next
end Node

object Node:
  def unapply[T <: scinear.Linear, O^](node: Node[T, O]^): (Box[T, O]^{node}, Box[Link[T, O], O]^{node}) =
    (node.elem, node.next)
end Node
```

Similar to the linear implementation, all structures are linear.
This is because `imem.Box` is linear and a field in all the internal structures' classes.

All class fields have an internal field, which starts with `_` and is passed as a constructor argument, and a corresponding external field.
The external field has the same value as the internal field, but its type captures `{this}`.
This approach for defining fields is necessary for [escape-checking](../background/capturing-types.md#escape-checking) to propagate to the class's fields and to prevent fields from escaping when an instance is bound to a scope.

The `LinkedList` class has two type parameters.
The first parameter is the element type `T`, which must be linear, similar to the linear implementation.
The second parameter is the owner capture set of the linked list instance, `O^`.
The class has a single field, `head`, which is a `Box` containing a `Link`, written as `Box[Link[T, O], O]`, and may point to the first node.
Both the `Box` and the `Link` are instantiated with `O`, which makes their lifetimes equal.

A `Link` is an `Option` containing a boxed `Node`, `Option[imem.Box[Node[T, O], O]]`.
This definition is equivalent to the one Rust version uses, except that `O` instantiates the `Owner^` type parameter of `Box` and the `O^` type parameter of `Node`.

In this version, as in other versions, a node has two fields. One field stores the element that the node references, and the other field points to the next node, if it exists.

The `elem` field of `Node` is a `Box` that points to a value of type `T`, with `O` as its owner, which is `Box[T, O]`.
This field represents a reference to an object of type `T`.

The `next` field of `Node` is a `Box` that points to a `Link`, with `O` as its owner, that is `Box[Link[T, O], O]`.

Finally, as in the linear version, since `Node` is a linear class with two fields, an `unapply` method is allows access to both fields of a node.

<!-- TODO: Move this to comparison -->
It is important to note the difference between Rust’s `List` and `Node` fields and their counterparts in imem.
The imem version wraps the fields in an additional `Box` compared to Rust.

This difference exists because, in Rust, holding a mutable reference to an object allows transitive mutation of its fields, which effectively makes those fields mutable.  
In imem, this behavior does not apply.

To support a similar mechanism, the imem guidelines advise wrapping every field in a `Box`.
Since boxes are the source of statically controlled mutability in imem, this approach allows the program to mutate the fields of an object when the program has a mutable reference to the object.

As a result, the implementation of user interface functions becomes slightly more complex.
<!-- TODO: Move this to comparison -->

## User Interface

To clarify the imem implementation and its details, this section describes each user interface function in turn.

The following presents the implementation of a helper function, `isEmpty`, which checks whether a list is empty:

```Scala
def isEmptyList[T <: scinear.Linear, @caps.use O1^, @caps.use O2^, WC^, MC^](
  self: ImmutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^
): Boolean =
  // access the `self`'s reference resource
  read[List[T, O1], O2, Boolean, {ctx}, WC, MC](self,
      list =>
        // new lifetime for borrowing the list's head
        val lf = Lifetime[{ctx, O2, O1}]()

        // borrow the list's head immutably, the new immutable reference will be `headRef`
        val (headRef, listHolder) = borrowImmutBox[Link[T, O1], O1, {ctx, O2}, lf.Key, lf.Owners, WC, MC](list.head)
        // access the head's (`headRef`'s) resource which is a `Link` to the next node
        // then check if the link whether is empty or not
        val isListEmpty = read[Link[T, O1], lf.Owners, Boolean, {ctx, O2}, WC, MC](headRef, link => link.isEmpty)

        // unlock the `listHolder`, to use both `lf` and `listHolder` linear variables
        unlockHolder(lf.getKey(), listHolder)
        // return the result
        isListEmpty
  )
```

First, the function signature includes five type parameters:

- `T`: the element type.
- `O1^`: the owner capture set of the list.
- `O2^`: the owner capture set of the immutable reference.
- `WC^`: the capture set that instantiates the `Context` write capability.
- `MC^`: the capture set that instantiates the `Context` move capability.

If `O1^` and `O2^` were the same type parameter, the function could not accept an immutable reference whose lifetime differs from the runtime of the list itself, which is an important use case.

Second, the function takes two arguments. The first argument, `self`, is an immutable reference to the list. The second argument is an implicit parameter, `ctx`, which is the imem `Context` instance.

The function is simple.
It accesses the list, which is the resource of `self` reference, through the `read` function.
Then, a new borrowing lifetime is defined.
Next, the function borrows the head of the list and accesses the link stored in the head’s `Box` to check whether it is empty.
Since the lifetime `lf` and the value holder `listHolder` are linear, the program must consume them, which it does using `unlockHolder`.

The following is the implementation of the push function:

```Scala
def push[T <: scinear.Linear, @caps.use O1^, @caps.use O2^ >: {O1}, @caps.use WC^, MC^](
  self: MutRef[List[T, O1], O2]^,
  elem: T
)(
  using ctx: Context[WC, MC]^
): Unit =
  // understand whether the list is empty:
  val (isListEmpty, self2) =
    // new lifetime for borrowing `self` immutably
    val lf = Lifetime[{ctx, O2}]()

    // re-borrow `self` immutably
    val (listRef, selfHolder) = borrowImmut[List[T, O1], O2, {ctx, O2}, lf.Key, lf.Owners, {WC}, {MC}](self)
    // checks if the list is empty
    val isListEmpty = isEmptyList(listRef)

    // return the result and unlock the `selfHolder` to get the `self` reference
    (isListEmpty, unlockHolder(lf.getKey(), selfHolder))

  // create a the new node that is going to be pushed:
  val newNode = Node(
    newBox[T, O1](elem), // node's element
    newBox[Link[T, O1], O1](None) // node's next link, which is initially `None`
  )

  if isListEmpty then
    // access the list mutably
    writeWithLinearArg[List[T, O1], {O2}, Unit, {ctx}, newNode.type, {WC}, {MC}](
      self2, // the mutable reference to the list
      newNode,
      ctx ?=> (list, newNode) =>
        // set the list's head to point to the new node
        setBox[Link[T, O1], {O1}, {ctx}, {WC}, {MC}](list.head, Some(newBox(newNode)))
    )
  else
    // access the list mutably
    writeWithLinearArg[List[T, O1], {O2}, Unit, {ctx}, newNode.type, {WC}, {MC}](
      self2,
      newNode,
      ctx ?=> (list, newNode) =>
        // create a temporary box pointing to a link that points to the new node
        val tempHead = newBox[Link[T, O1], O1](Some(newBox[Node[T, O1], O1](newNode)))
        // the state at here: {(temp head -> new node), (list -> previous first node), (new node -> None)}

        // swap the `tempHead` and the `list.head`, so that the `list.head` points to the new node
        val (tempHead2, listHead) = swapBox(tempHead, list.head)
        // the state at here: {(temp head -> previous first node) and (list -> new node), (newNode -> None)}

        // new lifetime for borrowing the list's head mutably
        val lf = Lifetime[{ctx, O1}]()

        // borrow the list's head mutably
        val (listHeadRef, listHeadHolder) = borrowMutBox[Link[T, O1], O1, {ctx}, lf.Key, lf.Owners, {WC}, {MC}](listHead)

        // access the `listHeadRef`'s resource, which is list's head, mutably
        writeWithLinearArg[Link[T, O1], lf.Owners, Unit, {ctx, O2}, tempHead2.type, {WC}, {MC}](
          listHeadRef,
          tempHead2,
          ctx ?=> (head, tempHead2) =>
            // get the head's box, that is pointing to the newly created node
            val nodeBox = head.get

            // new lifetime for borrowing the new node mutably
            val lfInner = Lifetime[{ctx, O1}]()

            // borrow the new node mutably
            val (nodeRef, nodeBoxHolder) = borrowMutBox[Node[T, O1], O1, {ctx}, lfInner.Key, lfInner.Owners, {WC}, {MC}](nodeBox)
            // access the `nodeRef`'s resource, which is the new node, mutably
            writeWithLinearArg[Node[T, O1], lfInner.Owners, Unit, {ctx}, tempHead2.type, {WC}, {MC}](
              nodeRef,
              tempHead2,
              ctx ?=> (node, tempHead2) =>
                // swap the new node's next and the temporary box
                swapBox[Link[T, O1], {O1}, {O1}, {ctx}, {WC}, {MC}](node.next, tempHead2)
                // the state at here: {(temp head -> None), (list -> new node), (new node -> previous first node)}
                // by here, the new node is pushed to the list
                ()
            )

            // unlock the `nodeBoxHolder`, to use both `lfInner` and `nodeBoxHolder` linear variables
            unlockHolder(lfInner.getKey(), nodeBoxHolder)
            ()
          )

        // unlock the `listHeadHolder`, to use both `lf` and `listHeadHolder` linear variables
        unlockHolder(lf.getKey(), listHeadHolder)
        ()
    )
```

The function type parameters are similar to those of `isEmptyList`, with a minor change: `O2` must be a superset of `O1`.
This requirement enables pushing by ensuring that the reference does not outlive the list.
The `push` function modifies some box's values, so the program signature has to annotate the `WC` type parameter with `@cap.use`.
The function also takes an additional argument, `elem`, which is the value of the new element that will be pushed onto the list.

The `push` implementation uses another imem interface, `writeWithLinearArg`.
This interface works like `write`, but it also allows the program to pass linear values to the `writeAction` via an explicit argument.
This is useful because an anonymous function passed as `writeAction` cannot use linear values available at `writeWithLinearArg`'s' call site's enclosing scope.
However, in some cases, the program needs to use such linear values, and it is acceptable for the program to expire them after calling `writeWithLinearArg`.

To push a new node onto the list, the program must update the box that the list head link points to, or in Scala terms, `list.head.get`, so that it points to the newly created node.
In addition, the newly created node must point to the node that was previously the first node in the list, which is the node referenced by the `list.head.get` box.

<!-- TODO: A DIAGRAM OF HOW THE BOXES SHOULD CHANGE DURING PUSHING STEP BY STEP (3 STEPS) -->

If the list is empty, only the first modification is enough, which is setting `list.head.get` to point to the new node.

If the list is not empty, an additional step is required.
First, the function creates a temporary head that points to the new node.
Then the `push` function swaps the temporary head with the list head.
Now the list points to the new node, and the temporary head points to the previously first node.
Finally, the function swaps the temporary head link with the newly created node’s `next` link.
This results in the list head pointing to the new node, and the new node pointing to the rest of the list from before the push.

The pop functionality implementation in imem is similar to push:

```Scala
def pop[T <: scinear.Linear, @caps.use O1^, @caps.use O2^ >: {O1}, @caps.use O3^ >: {O2}, @caps.use WC^, @caps.use MC^](
  self: MutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^
): Option[Box[T, O3]] =
  // understand whether the list is empty:
  val (isListEmpty, self2) =
    // new lifetime for borrowing `self` immutably
    val lf = Lifetime[{ctx, O2}]()

    // re-borrow `self` immutably
    val (listRef, selfHolder) = borrowImmut[List[T, O1], O2, {ctx, O2}, lf.Key, lf.Owners, {WC}, {MC}](self)
    // checks if the list is empty
    val isListEmpty = isEmptyList(listRef)

    // return the result and unlock the `selfHolder` to get the `self` reference
    (isListEmpty, unlockHolder(lf.getKey(), selfHolder))

  if isListEmpty then
    // converge if branches by consuming `self2`
    self2.consume()
    // return None, no element to pop
    None
  else
    // temporary box to hold the list's head
    val (tempHead, self3) =
      // new lifetime for borrowing `self2` mutably
      val lf = Lifetime[{ctx, O3}]()

      // borrow `self2` mutably
      val (listRef, selfHolder) = borrowMut[List[T, O1], O2, {ctx, O2}, lf.Key, lf.Owners, {WC}, {MC}](self2)

      // create the temporary box
      // the temporary box's lifetime capture set is `{O3}`, as it is going to be returned
      val tempHead = newBox[Link[T, O1], O3](None)
      // the state at here: {(temp head -> None), (list -> first node), (first node -> second node and the rest of the list)}

      // access the `listRef`'s resource, which is the list, mutably
      val tempHead2 = writeWithLinearArg[List[T, O1], lf.Owners, Box[Link[T, O1], O3], {ctx}, tempHead.type, {WC}, {MC}](
        listRef,
        tempHead,
        ctx ?=> (list, tempHead) =>
          // swap the `tempHead` and the `list.head`, so that the `tempHead` points to the first node
          val (newTempHead, listHead) = swapBox[Link[T, O1], {O3}, {O1}, {ctx}, {WC}, {MC}](tempHead, list.head)
          // the state at here: {(temp head -> first node), (list -> None), (first node -> second node and the rest of the list)}

          // `listHead` is a linear variable and have to be used
          // use `listHead` by consuming
          listHead.consume()
          // return the newly created temporary box pointing to the first node
          newTempHead
      )

      // return the temporary box and unlock the `selfHolder` to get the `self2` reference
      (tempHead2, unlockHolder(lf.getKey(), selfHolder))

    val tempHead2 =
      // new lifetime for borrowing `tempHead` mutably
      val lf = Lifetime[{ctx, O3}]()

      // borrow `tempHead` mutably
      val (tempHeadRef, tempHeadHolder) = borrowMutBox[Link[T, O1], O3, {ctx, O2}, lf.Key, lf.Owners, {WC}, {MC}](tempHead)

      // access the `tempHeadRef`'s resource, which is the link to the list's first node, mutably
      writeWithLinearArg[Link[T, O1], lf.Owners, Unit, {ctx}, self3.type, {WC}, {MC}](
        tempHeadRef,
        self3,
        ctx ?=> (head, self3) =>
          // new lifetime for borrowing the list's first node mutably
          val lfInner = Lifetime[{ctx, O1}]()

          // borrow the list's first node mutably
          val (firstNodeRef, firstNodeBoxHolder) = borrowMutBox[Node[T, O1], O1, {ctx}, lfInner.Key, lfInner.Owners, {WC}, {MC}](head.get)

          // access the `self3`'s resource, which is the list, mutably
          writeWithLinearArg[List[T, O1], {O2}, Unit, {ctx}, firstNodeRef.type, {WC}, {MC}](
            self3,
            firstNodeRef,
            ctx ?=> (list, firstNodeRef) =>

              // access the `firstNodeRef`'s resource, which is the first node, mutably
              writeWithLinearArg[Node[T, O1], lfInner.Owners, Unit, {ctx}, list.type, {WC}, {MC}](
                firstNodeRef,
                list,
                ctx ?=> (firstNode, list) =>
                  // swap the node's next and the list's head
                  swapBox[Link[T, O1], {O1}, {O1}, {ctx}, {WC}, {MC}](firstNode.next, list.head)
                  // the state at here: {(temp head -> first node), (list -> second node and the rest of the list), (first node -> None)}
                  ()
              )
          )

          // unlock the `firstNodeBoxHolder`, to use both `lfInner` and `firstNodeBoxHolder`
          unlockHolder(lfInner.getKey(), firstNodeBoxHolder)
      )

      // return the temporary box and unlock the `tempHeadHolder` to get the `tempHead` reference
      unlockHolder(lf.getKey(), tempHeadHolder)

    // in here, the `tempHead2` points to the popped first node and the list points to the rest of the list
    // the runtime memory state is correct but the lifetime capture set of the `tempHead2` still contains `{O1}`
    // `tempHead2`'s type is: `Box[Option[Box[Node[T, O1], O1]], O3]`

    // access the `tempHead2`'s resource, which is the link to the popped node
    derefForMoving[Link[T, O1], O3, {ctx}, Option[Box[T, O3]], {WC}, {MC}](
      tempHead2,
      link =>
        // box to the popped node
        val nodeBox = link.get
        // access the `nodeBox`'s resource, which is the popped node
        val movedElem = derefForMoving[Node[T, O1], O1, {ctx}, Box[T, O3], {WC}, {MC}](
          nodeBox,
          // move the box to popped node's element to the new owner
          node => moveBox[T, O1, {O3}, {WC}, {MC}](node.elem)
        )

        // return the moved element box
        Some(movedElem)
    )
```

The `pop` function introduces a new capture set type parameter, `O3^`, which represents the lifetime set associated with the popped element.
`O3^` must be a superset of `O2`, the reference lifetime set, because during the `pop` implementation there is a point where a reference with lifetime set `O3` reaches a reference with lifetime set `O2`.
After the element is popped, the program can freely move the `Box` to any lifetime, with no required relationship to either `O3` or `O2`.

Also, the function returns an `Option` to a `Box` to an element.
The `Box` lifetime capture set is `O3`.

To pop an element from the list, the program must update the box that the list head link points to, or in Scala terms, `list.head.get`, so that it points to the list's second node, and also it should return a box pointing to the first node's element.
To preserve the only one direct box requirement of imem memory well-formedness, the first node's next link should no longer point to the second node.

To pop an element from the list, the program must update the box that the head link points to, in Scala terms `list.head.get`, so that it points to the second node, and the function must return a box that points to the first node’s element.
To preserve the imem memory well-formedness requirement of having only one direct box, the first node’s next link must no longer point to the second node.

<!-- TODO: A DIAGRAM OF HOW THE BOXES SHOULD CHANGE DURING POPPING STEP BY STEP (3 STEPS) -->

If the list is empty, a pop operation returns `None`.

If the list is nonempty, the function first creates a temporary box and swaps it with the list’s head.
After this swap, the temporary box points to the first node, and the list’s head points to nothing.
Next, the `pop` function accesses the first node and swaps the first node’s box that points to the link that is pointing to the second node with the list’s head.
After this swap, the temporary box points to the first node, the list’s head points to the remainder of the list without the first node, and the first node points to nothing.

At this point, the node is popped, in runtime overview.
But the function needs to return a box that points to the element with the requested lifetime, `O3`.
To do so, the `pop` function dereferences the temporary box and then dereferences the first node to access and move the box that points to the first node’s element.
As a result, a box that points to the first node's element with the box's lifetime being `O3`, and the function returns it.

The `peek` function implementation in imem is the code that follows:

```Scala
def peek[T <: scinear.Linear, @caps.use O1^, @caps.use O2^, O3^, O4Key, @caps.use O4^ >: {O1, O2, O3}, WC^, MC^](
  self: ImmutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^{O3}
): Option[ImmutRef[T, O4]] =
  // access the `self`'s reference resource, which is the list
  read[List[T, O1], O2, Option[ImmutRef[T, O4]], {ctx}, {WC}, {MC}](self,
    ctx ?=> list =>
      // borrow the list's head immutably
      val (headRef, listHeadHolder) = borrowImmutBox[Link[T, O1], O1, {ctx}, O4Key, O4, {WC}, {MC}](list.head)

      // use `listHeadHolder` by consuming because it is a linear variable that is no longer needed
      listHeadHolder.consume()

      // access the head's (`headRef`'s) resource, which is a link to the next node
      read[Link[T, O1], O4, Option[ImmutRef[T, O4]], {ctx}, {WC}, {MC}](headRef,
        ctx ?=> linkOpt =>
          // without consuming the option, check whether `linkOpt` is empty or not
          val (linkOpt2, isListEmpty) = scinear.utils.peekLinearOption(linkOpt)

          // the behavior changes based on list emptiness
          if isListEmpty then
            // converge if branches by consuming `linkOpt2`
            linkOpt2.isEmpty
            // return None, no element to peek
            None
          else
            // get the box pointing to the list's first node
            val nodeBox = linkOpt2.get
            // borrow the first node immutably
            val (nodeRef, nodeBoxHolder) = borrowImmutBox[Node[T, O1], O1, {ctx}, O4Key, O4, {WC}, {MC}](nodeBox)
            // use `nodeBoxHolder` by consuming because it is a linear variable that is no longer needed
            nodeBoxHolder.consume()

            // access the `nodeRef`'s resource, which is the first node
            read[Node[T, O1], O4, Option[ImmutRef[T, O4]], {ctx}, {WC}, {MC}](
              nodeRef,
              ctx ?=> node =>
                // borrow the first node's element immutably
                val (nodeRef, nodeElemHolder) = borrowImmutBox[T, O1, {ctx}, O4Key, O4, {WC}, {MC}](node.elem)
                // use `nodeElemHolder` by consuming because it is a linear variable that is no longer needed
                // the function only needs the reference to the element
                nodeElemHolder.consume()
                // return the reference to list's first node's element
                Some(nodeRef)
            )
      )
  )
```

The `peek` function signature introduces two new type parameters: `O3^`, which is the `Context` lifetime capture set, and `O4^`, which is the lifetime set of the reference `peek` returns.
The result of `peek` is a reference to the first element of the list.
Therefore, the reference must expire before the list, before the reference to the list, and before all lifetimes aggregated by the `Context` up to this function.
For this reason, `O4^ >: {O1, O2, O3}` defines `O4^` as a superset of all these capture sets.

The implementation of the function is straightforward.
The `peek` function must borrow the box that points to the first node’s element.
To reach this box, it uses the `read` function to go through the reference to the list and access the list’s head.
Then, it borrows the list's head, follows the link to the first node, and, if the list is nonempty, reaches the box pointing to the first node.
Finally, it borrows the box pointing to the first node and accesses the box that points to the first node’s element.

The implementation of `peekMut` follows the same structure as `peek`:

```Scala
def peekMut[T <: scinear.Linear, @caps.use O1^, O2^, O3^, O4Key, @caps.use O4^ >: {O1, O2, O3}, @caps.use WC^, MC^](
  self: MutRef[List[T, O1], O2]
)(
  using ctx: Context[WC, MC]^{O3}
): Option[MutRef[T, O4]] =
  // access the `self`'s reference resource, which is the list
  write[List[T, O1], O2, Option[MutRef[T, O4]], {ctx}, {WC}, {MC}](self,
    ctx ?=> list =>
      // borrow the list's head mutably
      val (headRef, listHeadHolder) = borrowMutBox[Link[T, O1], O1, {ctx}, O4Key, O4, {WC}, {MC}](list.head)

      // use `listHeadHolder` by consuming because it is a linear variable that is no longer needed
      listHeadHolder.consume()

      // access the head's (`headRef`'s) resource, which is a link to the next node
      write[Link[T, O1], O4, Option[MutRef[T, O4]], {ctx}, {WC}, {MC}](headRef,
        ctx ?=> linkOpt =>
          // without consuming the option, check whether `linkOpt` is empty or not
          val (linkOpt2, isListEmpty) = scinear.utils.peekLinearOption(linkOpt)

          // the behavior changes based on list emptiness
          if isListEmpty then
            // converge if branches by consuming `linkOpt2`
            linkOpt2.isEmpty
            // return None, no element to peek
            None
          else
            // get the box pointing to the list's first node
            val nodeBox = linkOpt2.get
            // borrow the first node mutably
            val (nodeRef, nodeBoxHolder) = borrowMutBox[Node[T, O1], O1, {ctx}, O4Key, O4, {WC}, {MC}](nodeBox)
            // use `nodeBoxHolder` by consuming because it is a linear variable that is no longer needed
            nodeBoxHolder.consume()

            // access the `nodeRef`'s resource, which is the first node
            write[Node[T, O1], O4, Option[MutRef[T, O4]], {ctx}, {WC}, {MC}](
              nodeRef,
              ctx ?=> node =>
                // borrow the first node's element mutably
                val (nodeRef, nodeElemHolder) = borrowMutBox[T, O1, {ctx}, O4Key, O4, {WC}, {MC}](node.elem)
                // use `nodeElemHolder` by consuming because it is a linear variable that is no longer needed
                // the function only needs the reference to the element
                nodeElemHolder.consume()
                // return the mutable reference to list's first node's element
                Some(nodeRef)
            )
      )
  )
```

The implementation of the `peekMut` function closely follows that of `peek`.
The differences are as follows:

- The `peekMut` function annotates the `WC^` type parameter with `@caps.use` because it calls `borrowMutBox` in its body.
- The `peekMut` function uses `write` and `borrowMutBox` instead of the `read` and `borrowImmutBox` functions used by `peek`.

