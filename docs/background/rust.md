# Rust

[Rust](https://doc.rust-lang.org/book/title-page.html) is a programming language that provides static memory management and explicit mutability control through [ownership rules](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) and the [borrow checker](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html).
This section provides a brief introduction to Rust’s static ownership rules and borrow checking.

## Ownership

Rust's enforces the following static ownership rules:

- Each value in Rust has a variable that’s called its owner.
- There can only be one owner at a time.
- When the owner goes out of scope, the value will be dropped.

In the following example, `s1` is the owner of its value, which is a string `"hello"` in the heap:

```Rust
{
  let s1 = String::from("hello");
} // s1's value is (freed) dropped here
```

During assignment, except for trivial types such as integers and other copyable types, Rust transfers ownership of the right-hand side value to the left-hand side of the assignment.
This ownership transfer is called a move.
As a result, values in Rust behave similarly to linear values in [linear type](./linear-types.md) systems; when they are passed to functions or assigned to other variables, access to the original variable is lost.

```Rust
let s1 = String::from("hello");
let s2 = s1; // s1 is moved to s2

println!("{}, world!", s1); // error
```

In the examples above, the ownership of the value that `s1` points to is transferred from `s1` to `s2`.

### Box Pointer

Rust provides `Box<T>` pointers that allow a program to create pointers to values whose size is not known statically.
These values reside on the heap.
Such pointers are useful for defining recursive types, as in the following example:

```Rust
enum List {
    Cons(i32, List), // error
    Nil,
}

enum List {
    Cons(i32, Box<List>), // ok
    Nil,
}
```

The ownership of `Box<T>` pointers is similar to that of other values.
They have a single owner, which is a variable, at each point during program execution, and when the owner goes out of scope, the pointer and the value it points to are dropped (freed).

## Borrowing 

------------------------------------------

<!-- TODO: This might be in the background, but you should have an explanation somewhere of what a Box means/does in Rust. -->

# Ownership

TODO: Explain rust style ownership and borrowing, most of them is about ownership but Rust stuff in examples and at the end. This would replace the rust section

This section begins with a broad overview of different kinds of ownership, followed by the thesis's focus on variables that own their value. It mentions that Rust uses this concept, and a couple of examples in Rust and Scala (using `imem`) are shown. Then, it elaborates on the memory management use case of this concept, which is that a value can be freed when its owner is out of scope.

ONLY SAY THE STUFF THAT THE READER NEEDS

## Borrowing

It describes what borrowing is and why it is useful in the first place, and how it helps the programmer to define functions and classes without returning every argument that is needed. Then it explains the borrowing rules:
- There should be either one mutable or multiple immutable borrowed references to an object at the same time.
- No borrowed reference should outlive the main object.

## Stacked borrows model

A brief explanation of [stacked borrowed model](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf) with a simple example.
