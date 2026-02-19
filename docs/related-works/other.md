# Outside Scala

Static memory management, along with static control of aliasing and mutability, is a well-known research problem that has been addressed by many projects.
Approaches that tackle these challenges, fully or partially, often also aim to support fearless concurrency and guarantee statically data-race-free programs.
The following section highlights some of these approaches outside the Scala ecosystem and discusses their relation to imem.

[Reachability Types](https://dl.acm.org/doi/epdf/10.1145/3485516) enable ownership-style reasoning in higher-order functional languages by introducing a type system that tracks potential aliasing variables within a value’s type.
The system associates reachability sets with types, providing a selective separation mechanism that controls how values may alias one another.
This design supports move semantics, ownership, and safe parallelism, while imposing fewer restrictions than Rust’s aliasing rules.
Although this approach targets goals similar to those of imem, it introduces an entirely new type system.
In that sense, it is closer to capture checking than to imem’s approach, which aims to demonstrate that these goals are feasible within the scope of a library.

[Affe](https://dl.acm.org/doi/epdf/10.1145/3408985) is an extension of the ML programming language that adds support for linearity and alias management.
Affe annotates types with kinds (which can be linear, affine, or unrestricted), and allows borrowing of linear values in both exclusive mutable and shared immutable ways.
It does not require heavy annotations for linearity and provides fully automatic type and region inference.
Affe targets goals similar to those of imem, and its expressive yet easy-to-use approach to linearity could inspire Scinear.
However, Affe introduces a new type system, making it harder to integrate with existing programs, compared to imem, which is a library.

[MLKit](https://elsman.com/pdf/mlkit-4.7.16.pdf) is a compiler for Standard ML that provides fully automatic region inference without imposing restrictions on programs.
Its static region inference guarantees safe memory management, ensuring the absence of dangling references.
However, this fully automatic and unrestricted approach can cause some regions to outlive their actual lifetimes, leading to growing regions and potential memory leaks.
To address this issue, MLKit offers both safe and unsafe program-level APIs to check whether a region’s memory can be reset.
It also includes a minimal garbage collector that tracks regions.
MLKit does not target alias or mutability control and does not provide ownership or borrowing mechanisms.
[OwnML](https://github.com/amsen20/OwnML) is an extension to the MLKit compiler that aims to invalidate programs with memory leaks.
It does so by ensuring that some variables own their regions, enforcing separation from other values in their scope.
This limits cases where the region inference algorithm would otherwise merge objects with different lifetimes, which can lead to memory leaks.

[Pony]()