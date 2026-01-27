# Evaluation

This chapter focuses on building a practical and commonly used data structure, a linked list, using imem.
The linked list presented in this chapter is a simple polymorphic stack that provides the following interfaces to the user:

- checking whether the list is empty.
- pushing an item onto the stack.
- popping an item from the stack.
- peeking at the first item without popping it.
- iterating over the stack.

The main idea behind the linked list implementation is to avoid exposing the internal list representation to the user.
Therefore, all user-facing functions and methods return only linked list objects, element objects, or references, and never the internal data structures used by the list.

To evaluate the performance and practicality of imem, and to clarify what imem enables in practice, the imem implementation is compared against the following baselines:

- ***Rust:***
  A minimal implementation in Rust is used as Rust is the main inspiration for imem’s design.
  This baseline allows a direct comparison between imem’s ownership and borrow checking and Rust’s corresponding mechanisms.
  The implementation follows the approach presented in the tutorial [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/index.html)'s [*An Ok Stack*](https://rust-unofficial.github.io/too-many-lists/second.html).
- ***Vanilla Scala:***
  A simple Scala implementation that demonstrates how the data structure would look like in Scala without ownership or mutability control.
  This version is largely a direct translation of the Rust implementation into Scala.
- ***Linear Scala:***
  This implementation aims to provide as much functionality as possible while strictly adhering to linearity, [Scinear](../scinear/index.md), rules.
  The comparison between this version and the imem implementation illustrates the practical advantages of imem over a purely linear approach.

