# Static memory management library `imem`

imem is a memory management library that ensures resources adhere to a minimal version of the [Stacked Borrows Model](../background/stacked-borrows.md).
It achieves this goal through two distinct and independent mechanisms: static enforcement and dynamic verification.

During runtime, imem monitors each resource accessed by different references to ensure that none violate the Stacked Borrows Model rules.
This runtime verification is optional, so it can be disabled to prioritize performance.

During compilation, imem utilizes Scala type system features, including [capture checking](../background/capturing-types.md), [dependent types](../background/dependent-types.md), and linear types provided by the [scinear plugin](../scinear/index.md), to implement ownership and borrow checking.
This approach is similar to Rust's ownership system and borrow checker.
However, imem is a library rather than a compiler component.
imem's static enforcement is not entirely sound and contains loopholes; the [soundness section](soundness.md) discusses these loopholes and suggests minimal guidelines to avoid them.

Additionally, imem does not manage the actual allocation and deallocation of memory.
The [future works section](../conclusion/future-works.md) sketches how imem can be connected to actual memory management in Scala Native.
