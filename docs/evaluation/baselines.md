# Base lines

## Rust

This Rust implementation follows the approach described in [here](https://rust-unofficial.github.io/too-many-lists/second.html).

### Internal Structures

Internal structures of the linked list and their definitions are as follows:

```Rust
pub struct List<T> {
  head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
  elem: T,
  next: Link<T>,
}
```

The implementation simply models the list as a sequence of nodes linked together.
Each node contains an element and a reference to the next node.
A link may be `None`, which indicates that there is no subsequent node.

It is important to note that `Link<T>` is defined as an optional `Box` that refers to a node.
As a result, each node owns the next node through a `Box`, which causes the entire list to be owned by the first element.

The `List<T>` contains a link to the first node if the list is not empty.
Any variable bound to this value therefore owns the entire list.

In Rust, values are [moved](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) by default rather than copied.
As a result, `List`, `Node`, and `Link` instances are moved when they are not accessed through references, and they therefore behave like linear values.

### User Interface

In Rust, push, pop, and peek implementations are the following:

```Rust
impl<T> List<T> {

  pub fn new() -> Self {
    List { head: None }
  }

  pub fn push(&mut self, elem: T) {
    let new_node = Box::new(Node {
      elem: elem,
      next: self.head.take(),
    });
    self.head = Some(new_node);
  }

  pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|node| {
      self.head = node.next;
      node.elem
    })
  }

  pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
  }

  pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| &mut node.elem)
  }

}
```

The `new` function just creates a list with no node.

The `push` function takes a mutable reference to the list and an element.
It first creates a new node for the given element.
While constructing this node, the call to `self.head.take()` replaces the current value of `self.head` with `None` and returns the previous value.
Finally, a new `Box` is created for the node, and `self.head` is updated to point to this new box.

In the `pop` function, `self.head` is first swapped with `None`, and the resulting `Option` is then mapped.
If the list is non-empty, execution enters the function passed to `map`.
This function updates `self.head` to the next node of the previously first node.
Then it returns the element stored in the popped node, which results in moving that value.

The `peek` and `peek_mut` functions both return an option, because the list might be empty, containing a reference to the first node's element.
The returned reference is immutable in `peek` and mutable in `peek_mut`.
The peeking functions, borrow a reference to the first node, if it exists, using `as_ref` or `as_mut`, and then dereference the reference and borrow the node's element.
This dereferencing and borrowing happens in `&node.elem`.

The `peek` and `peek_mut` functions both return an option, since the list may be empty.
When present, the option contains a reference to the first node's element.
The reference returned by `peek` is immutable, whereas `peek_mut` returns a mutable reference.
Both functions borrow a reference to the first node, if it exists, by using `as_ref` and `as_mut`.
They then dereference this node reference and borrow its element, which occurs in the expression `&node.elem` and `&mut node.elem`.

### Iterators

In evaluation, all versions try to implement two kinds of iterators: a consuming iterator and a mutable iterator.

#### Consuming Iterator

A consuming iterator traverses the list by taking ownership of it, which means the list is moved into the iterator and stored as its fields.
The following shows the implementation of the consuming iterator in Rust:

```Rust
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {

  pub fn into_iter(self) -> IntoIter<T> {
    IntoIter(self)
  }

}

impl<T> Iterator for IntoIter<T> {

  type Item = T;

  fn next(&mut self) -> Option<Self::Item> {
    // access fields of a tuple struct numerically
    self.0.pop()
  }
}
```

The `IntoIter` structure contains a field that holds the list.
The `into_iter` function takes ownership of the list by moving as an argument, not getting it as a reference, and stores it inside the consuming iterator, `IntoIter<T>`.  
During iteration, the `next` method repeatedly pops elements from the list, one at a time.

#### Mutable Iterator

A mutable iterator borrows the list mutably, and unlike a consuming iterator, it does not change the list ownership.
The following is the mutable iterator implementation:

A mutable iterator borrows the list mutably and, unlike a consuming iterator, does not take ownership of the list.
The following presents the implementation of the mutable iterator:

```Rust
pub struct IterMut<'a, T> {
  next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {

  pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut {
      next: self.head.as_deref_mut(),
    }
  }

}

impl<'a, T> Iterator for IterMut<'a, T> {

  type Item = &'a mut T;

  fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
      self.next = node.next.as_deref_mut();
      &mut node.elem
    })
  }

}
```

The `IterMut<'a, T>` structure contains a field, `next`, which stores an optional mutable reference to the next node that the iterator should visit.
It is an `Option` because, once iteration reaches the end of the list, there is no node left to visit and the value becomes `None`.

The structure is parameterized by the lifetime `'a` because the iterator holds a reference into the list.
Therefore, the iterator must not outlive the mutable borrow of the list.
The `'a` lifetime makes this relationship feasible, and it informs the compiler that the reference lifetime can differ from one iterator instance to another.

The `iter_mut` function creates a mutable iterator reference.
Its return type is `IterMut<'_, T>`, which instructs the compiler to replace `'_` with the only other lifetime in the function signature, the lifetime of the `&mut self` reference.
The function then constructs an `IterMut` instance by dereferencing `self.head` and borrowing it mutably using `as_deref_mut`, if `self.head` is not `None`.

The `next` method first swaps `self.next` with `None` and retrieves the previous value.
If this value is not `None`, execution enters the function passed to `map`.
At this stage, the method dereferences the reference to the next node and borrows it mutably.
It then returns a mutable reference to the element stored in the current node.
The returned reference has the same lifetime as the iterator itself.

It is important to note that although the mutable iterator yields mutable references to elements, the list's structure does not change during iteration.
Furthermore, the program can use the list elsewhere and reborrow it mutably or immutably, which causes the reference in the iterator's field to expire and, consequently, the iterator to expire.

## Vanilla Scala

This subsection focuses on translating the Rust implementation into Scala and does not follow a pure functional programming model.
In Scala, references are managed by a garbage collector, both on JVM and native.
As a result, this implementation assumes no ownership relationship between variables and references, and it also assumes that reference lifetimes are not known statically.

Furthermore, as of Scala version 3.7.3, there is no non‑experimental support for compile‑time mutability control.
Therefore, this implementation does not restrict the mutability of objects that are reachable from an element in the list.
Instead, it only provides interfaces that allow the element itself to be updated by replacing the reference currently stored in the list with another reference.

### Internal Structures

The internal structures mirror the Rust implementation:

```Scala
class List[T](var head: Link[T] = None)

type Link[T] = Option[Node[T]]

class Node[T](var elem: T, var next: Link[T])
```

Similar to the Rust version, the list is modeled as a sequence of nodes linked together.
The `List` structure holds a link that may be empty and that refers to the first node.

Each node contains an element and a link to the next node, which may or may not exist.

In this implementation, the `List.head` and `Node.next` fields are mutable, because the structure of the list can change during execution.
The `Node.elem` field is also mutable, since updating an element is performed by replacing the reference to the stored object with a new reference.

### User Interface

The user interfaces are defined as follows:

```Scala
class List[T](var head: Link[T] = None):

  def push(elem: T): Unit = {
    val newNode = new Node(elem, this.head)
    this.head = Some(newNode)
  }

  def pop(): Option[T] = {
    this.head.map { node =>
      this.head = node.next
      node.elem
    }
  }

  def peek: Option[T] = {
    this.head.map(_.elem)
  }

  def peekMut(): Option[Node[T]] = {
    this.head
  }

  ...

end List
```

The overall logic of this implementation matches the Rust version, except that it does not include ownership or borrow checking.
One notable difference is that `peekMut` returns an `Option[Node[T]]`, which conflicts with the goal of hiding internal data structures from the user.
However, this design choice is necessary, because there is no alternative way to return a value such that modifying the element also updates the element stored in the list.

### Iterators

#### Consuming Iterator

The following illustrates how a consuming iterator is implemented in this Scala version of the linked list:

```Scala
class List[T](var head: Link[T] = None):

  ...

  def intoIter(): Iterator[T] = new Iterator[T] {
    def hasNext: Boolean = head.isDefined
    def next(): T = pop().getOrElse(throw new NoSuchElementException("next on empty iterator"))
  }

  ...

end List
```

Similar to the Rust version, this iterator pops elements from the list as it iterates.
When the iterator reaches the end of the list, the `Iterator[T]` trait requires the iterator to throw an exception.
Throughout the iteration, the iterator yields the elements themselves.

#### Mutable Iterator

```Scala
class List[T](var head: Link[T] = None):

  ...

  def iterMut(): Iterator[Node[T]] = new Iterator[Node[T]] {
    private var current: Link[T] = head
    def hasNext: Boolean = current.isDefined
    def next(): Node[T] = {
      current match {
        case Some(node) =>
          current = node.next
          node
        case None =>
          throw new NoSuchElementException("next on empty iterator")
      }
    }
  }

  ...

end List
```

Due to lack of borrowing and reference mutability control in this version, the mutable iterator must yield `Node[T]` values.
In this design, updating a node directly updates the corresponding element in the list.
The remainder of the implementation is straightforward.

It is important to note that this approach does not produce compile‑time errors when multiple iterators traverse the list simultaneously or when elements are pushed to or popped from the list during iteration.

## Linear Scala

The Linear Scala implementation of the linked list serves as a baseline for evaluating the practicality of imem.
This implementation uses the [Scinear](../scinear/index.md) plugin to enforce linearity rules.

This implementation focuses on the case where the elements themselves are linear, and it does not bypass linearity rules using the `@HideLinearity` annotation.
A non-linear type can still be an element of the list with the help of a linear wrapper.

### Internal Structures

The internal structures are quite similar to the Rust and vanilla Scala implementations:

```Scala
class List[T <: Linear](val head: Link[T]) extends Linear

type Link[T <: Linear] = Option[Node[T]]

class Node[T <: Linear](val elem: T, val next: Link[T]) extends Linear


// `unapply` functions to destruct a node into its element and its link to the next node:
object Node:
  def unapply[T <: Linear](node: Node[T]): Option[(T, Link[T])] =
    Some((node.elem, node.next))
end Node
```

As in the other baselines, this version's implementation of linked list is a chain of nodes connected to one another.
Each node contains an element and a link to the next node.
A link is an `Option[Node[T]]`, and the list itself holds a link to the first node, if one exists.

All structures are linear, because their fields have linear types, and `Link[T]` is an `Option` that is promoted to a linear type.
To allow access to both fields of a `Node`, an `unapply` method is implemented for the `Node` type.

### User Interface

Because of Scinear rules, this version defines the user interface as functions rather than methods:

```Scala
def push[T <: Linear](list: List[T], elem: T): List[T] =
  val newNode = Node(elem, list.head)
  List(Some(newNode))

def pop[T <: Linear](list: List[T]): (Option[T], List[T]) =
  list.head match
    case Some(Node(elem, next)) =>
      (Some(elem), List(next))
    case None =>
(None, List(None))
```

Because the elements are linear, it is not possible to implement `peek` or `peekMut` without violating linearity rules.
In linear memory, linear objects form a tree structure.
The return value of a `peek`/`peekMut` operation would refer to an element while the list node still exists, which breaks this tree structure.

The `push` function takes a list instance and returns a modified list with the new element prepended.
It first creates a new node whose `next` field points to the current first node of the list.
It then constructs a new list that points to this newly created node.

Similarly, the `pop` function takes a list as input and returns a modified list.
In addition, it returns an option containing the removed element.
The function checks whether `list.head` is `None`.
If so, no element is removed.
Otherwise, it returns the first node's element and creates a new list that points to the former node’s `next` link.

### Iterators

#### Consuming Iterator

Because a consuming iterator takes ownership of the list by consuming it, the iterator can be implemented without violating linearity rules:

```Scala
def intoIter[T <: Linear](list: List[T]): IntoIter[T] = IntoIter(list)

class IntoIter[T <: Linear](val list: List[T]) extends Linear

def hasNext[T <: Linear](iter: IntoIter[T]): (IntoIter[T], Boolean) =
  val (head, notHasNext) = scinear.utils.peekLinearOption(iter.list.head)
  (IntoIter(List(head)), !notHasNext)

def next[T <: Linear](iter: IntoIter[T]): (Option[T], IntoIter[T]) =
  val (elem, nextList) = pop(iter.list)
  (elem, IntoIter(nextList))
```

As in the other implementations, the `IntoIter` structure simply stores the list as one of its fields.
Both `IntoIter` and `List` are linear types, which ensures that no other object can reference the `List` instance once it is passed to `IntoIter`.

The `hasNext` function checks whether the list’s head is empty by using `scinear.utils.peekLinearOption`.
Scinear provides this to allow the program to inspect whether an option is `None` without consuming its value.
After this check, `hasNext` reconstructs the iterator and the list using the retrieved head node option.

The `next` function calls `pop` to remove the first element of the list, if one exists.
It then returns the popped element and constructs a new iterator pointing to the list produced by the `pop` operation.

#### Mutable Iterator

Similar to the `peek` and `peekMut` functions, mutable iterators are incompatible with linearity rules.
A linear mutable iterator would need to hold a reference to the list while another reference to the same list already exists, which would violate linear memory well‑formedness by allowing multiple linear objects to point to the same linear value.

