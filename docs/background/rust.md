# Rust

[Rust](https://doc.rust-lang.org/book/title-page.html) is a programming language that provides static memory management and static mutability control through [ownership rules](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) and the [borrow checker](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html).
This section provides a brief introduction to Rust’s static ownership rules and borrow checking.

## Ownership

Rust enforces the following static ownership rules:

- Each value in Rust has a variable that’s called its owner.
- There can only be one owner for each value at a time.
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

To allow a program to access a value without taking ownership of it, that is, without moving it, Rust provides borrowing.
A program can borrow a value and create references to it.
Borrowing can be mutable or immutable, which results in either a mutable reference or an immutable reference.
The difference between holding a reference to a value and owning it directly is that a reference does not own the value.
When the reference goes out of scope, only the reference is dropped, and the value it points to is not dropped.
The following program demonstrates a Rust example that borrows a string value, first mutably and then immutably:

```Rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s; // borrow `s` mutably
    r1.push_str(" world"); // update the string

    let r2 = &s; // borrow `s` immutably
    println!("Immutable borrow: {}", r2); // print the string
}
```

A lifetime represents a statically known consecutive part of a program, which is also referred to as a non-lexical scope.
Every reference has a lifetime, which is the part of the program during which the reference is valid.
The following code shows the previous example with annotated lifetimes:

```Rust
fn main() {
    let mut s = String::from("hello");    // -----------------+-- 'a
                                          //                  |
    let r1 = &mut s;                      // -+-- 'b          |
    r1.push_str(" world");                // -+               |
                                          //                  |
    let r2 = &s;                          // -+-- 'c          |
    println!("Immutable borrow: {}", r2); // -+               |
}                                         // -----------------+
```

In the example above, the lifetimes of both references, `r1` and `r2`, which are `'b` and `'c`, lie within the non-lexical scope in which `s` owns the string, namely `'a`.
In addition, the lifetime `'b` of the mutable reference `r1` ends before the lifetime `'c` of the immutable reference `r2` begins.

The borrow checker ensures the following properties by reasoning about reference lifetimes statically:

- At any given time, the program can have either one mutable reference or any number of immutable references to a value, which limits aliasing.
- The value that a reference points to should not be dropped, meaning that there are no dangling pointers.

The first goal is to ensure static mutability control by preventing the existence of a mutable reference together with another mutable reference or with immutable references at the same time.
In the example above, at the point where the lifetime of the immutable reference `r2` begins, the lifetime of the mutable reference `r1` ends.

The second goal of the borrow checker is to ensure that no dangling pointers exist at any point during execution.
To achieve this goal, Rust enforces a stricter rule regarding reference lifetimes: a reference lifetime must lie within the non-lexical scope in which the owner of the value at the time of borrowing remains the owner of that value.
Therefore, if a value is borrowed and a reference is created, and the value is then moved, rather than dropped, the reference becomes invalid and its lifetime ends, as the example below shows:

```Rust
fn main() {
    let s1 = String::from("Hello"); // s1 is the Owner
    let r1 = &s1; // r1 borrows s1

    let s2 = s1; // both s1 and r1 become invalid

    println!("{}", r1); // error
}
```
