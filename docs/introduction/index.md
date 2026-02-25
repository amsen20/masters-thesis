# Introduction

Rust is popular in systems programming because safe memory management and alias control are built into the language.
Static memory management removes the need for garbage collectors, which can introduce performance problems.
Garbage collection ranges from stop-the-world pauses to more complex approaches such as generational garbage collection.
However, these approaches can still slow programs at runtime in ways that are not fully controllable by the programmer.
Therefore, systems programmers often prefer languages that are not garbage collected.

Removing garbage collection can improve performance, but it also increases the risk of unsafe memory management.
Unsafe memory management can lead to use-after-free, double-free, and memory leaks.
Manual memory management, as used in C and C++, is vulnerable to these errors.
Rust addresses these problems through ownership rules and borrowing, which limit the set of valid programs and enforce memory safety statically.

In addition, alias and mutability control enables features such as fearless concurrency, aggressive compiler optimizations, and safe iteration.
Rust provides alias and mutability control through its borrow checker.

Scala is a language that supports both object-oriented and functional programming.
Alongside that, Scala also provides a safe and strict static type system with strong theoretical foundations.
In 2016, Scala also entered systems-level programming by supporting a [native](https://scala-native.org/en/latest/index.html) output backend that emits LLVM IR.

Scala historically relies on the JVM and its garbage collection for memory management.
Scala Native instead relies on configurable third-party garbage collectors designed for LLVM languages, namely [Boehm-Demers-Weiser](https://www.hboehm.info/gc/) and [Immix](https://www.steveblackburn.org/pubs/papers/immix-pldi-2008.pdf).

The motivating [example](./motivation.md) demonstrates that these garbage collectors, combined with functional programming paradigms such as immutability, can perform poorly without high-level language support.
Furthermore, even a well-configured garbage collector can still have performance problems that static memory management would resolve.
Also, Scala Native users currently do not have the alias and mutability control that Rust provides.

imem enables Rust-style ownership rules and borrow checking in Scala as only a library.
To support this design, imem also depends on a minimal compiler plugin, [Scinear](../scinear/index.md), which adds [linear types](../background/linear-types.md) to Scala.
imem illustrates that the Scala type system, together with the recent addition of [capture checking](../background/capturing-types.md), is expressive enough that linearity is the only remaining feature needed, and that all the rest of Rust-style ownership and borrow checking can be implemented as a library.

This thesis presents the following:

- **Scinear:**
  A minimal compiler plugin that adds linear types to Scala and enforces linearity rules.
  The plugin also mixes linearity with capturing types.

- **imem:**
  A library that implements Rust-style ownership and borrow checking.
  imem defines boxes and immutable and mutable references, and it provides borrowing interfaces to borrow boxes immutably and mutably.
  The library enforces static memory-safety rules and alias and mutability control in the portion of memory that all boxes point to.

- A case study that implements safe linked lists to demonstrate imem capabilities and compare imem with Rust, vanilla Scala, and Scala with linearity rules.
