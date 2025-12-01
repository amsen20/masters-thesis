# Static memory management library `imem`

imem is a memory management library that ensures resources adhere to a minimal version of the [Stacked Borrows Model](/background/stacked-borrows) through static enforcement and dynamic verification.

During compilation, imem utilizes Scala type system features, including [capture checking](/background/capturing-types), [dependent types](/background/dependent-types), and linear types provided by the [scinear plugin](/scinear/), to implement ownership and borrow checking. This approach is similar to Rust and results in the stacked borrows model. However, this enforcement is not entirely sound and contains loopholes; [soundness section](/imem/soundness) discusses these loopholes and suggests minimal guidelines to avoid them.

During runtime, imem monitors each resource accessed by different references to ensure that none violate the stacked borrows rules. This runtime verification is optional, so it can be disabled to prioritize performance.

Additionally, imem does not manage the actual allocation and deallocation of memory. [Future works section](/conclusion/future-works) sketches how imem connects to actual memory management in Scala Native.
