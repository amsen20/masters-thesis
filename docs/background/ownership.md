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
