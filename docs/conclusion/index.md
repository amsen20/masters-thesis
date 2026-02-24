# Conclusion

This thesis outlined [Scinear](../scinear/index.md), [imem](../imem/index.md), and a [case study](../evaluation/index.md) that demonstrates imem capabilities.

The thesis presented that adding linear types to the Scala type system, and combining linearity with polymorphism and capture checking, lays the foundation for a library that enforces ownership and borrowing rules statically.
This approach avoids extending the type system with features that target ownership and borrowing as a special case.
Therefore, enabling Rust-style static memory management and alias control in Scala does not require foundational changes to the type system, and it can be implemented as a library.

To support this goal, the thesis explains the Scinear plugin rules for Scala, how Scinear mixes linearity with polymorphism and capture checking, and how Scinear is implemented.
Furthermore, the thesis discusses the imem formal memory model, how imem uses the Scala type system alongside Scinear to enforce ownership and borrowing rules as a library, and the limitations of this approach.
Finally, the thesis demonstrates imem by implementing a safe linked list.
A comparison with Rust, vanilla Scala, and linear Scala shows that imem is as expressive as Rust, but imem needs additional features to reduce verbosity.

The remaining work that can make imem more capable and easier to use is discussed in the next section.
