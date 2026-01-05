# Memory management library `imem`

imem is a static memory management library that ensures programs safely access and modify resources tracked by imem, while guaranteeing that reference accesses adhere to a minimal form of the [Stacked Borrows Model](../background/stacked-borrows.md).
The formal basis of imem’s static memory management is described in the [memory management](./memory-management.md) section.
In addition to static enforcement, imem also utilizes dynamic verification to inform the user when workarounds to the static rules lead to potential safety violations.

During [runtime](./runtime.md), imem monitors each resource accessed by different references to ensure that none violate the Stacked Borrows Model rules.
This runtime verification is optional, so it can be disabled to prioritize performance.

During compilation, imem utilizes Scala type system features, including [capture checking](../background/capturing-types.md), [dependent types](../background/dependent-types.md), and linear types provided by the [scinear plugin](../scinear/index.md), to implement [ownership](./ownership.md) and [borrow checking](./borrow-checking.md).
This approach is similar to Rust's ownership system and borrow checker.
However, imem is a library rather than a compiler component.
imem's static enforcement is not entirely sound and contains loopholes; the [soundness section](soundness.md) discusses these loopholes and suggests minimal guidelines to avoid them.

Additionally, imem does not manage the actual allocation and deallocation of memory.
The [future works section](../conclusion/future-works.md) sketches how imem can be connected to actual memory management in Scala Native.

