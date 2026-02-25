# Future Works

The imem formal foundation includes a memory model definition and a well-formedness condition.
However, imem soundness is not proved.
A formal soundness proof for imem should start from a minimal foundation that represents Scala extended with the Scinear plugin.
This foundation can be the \(CC_{<:\Box}\) calculus, described in [capturing types](https://dl.acm.org/doi/10.1145/3618003), extended with linear types.
Alternatively, it can be a typed lambda calculus that includes linear types in its type system and includes escape checking and capture set type parameters as well-defined features.
In addition, the proof should target imem correctness for safe static memory management, which implies no use-after-free and no double-free.
The proof also has to address alias and mutability control by proving mutual exclusion of mutable access to resources.

imem implementations tend to be long, and they can include loopholes.
Both issues can be mitigated by using Scala meta programming, as described in the [foundational paper](https://dl.acm.org/doi/abs/10.1145/2489837.2489840) and in the [Scala 3 guide](https://docs.scala-lang.org/scala3/guides/migration/compatibility-metaprogramming.html), and by adding more helper functions to the imem interface.
Scala macros can enforce common patterns.
For example, macros can enforce the borrowing pattern that introduces a lifetime `Owner` and `Key` type members in borrowing-interface calls, which makes avoiding loopholes easier.
In addition, small but useful operations such as writing, setting, and swapping a box resource directly, without repetitive borrowings, can significantly reduce duplication in codebases that use imem.
Similarly, a dereferencing operation that is similar to `read`, but for a box, can further reduce boilerplate.

In addition, imem currently does not manage memory allocation at runtime.
To address runtime allocation, the `nativelib` implementation in [Scala Zones](https://github.com/scala-native/scala-native/pull/3120) paves the way for runtime memory management in the native target.
Also, the Region-based off-heap memory for Scala [report](https://www.researchgate.net/profile/Denys-Shabalin/publication/291105865_Region-based_off-heap_memory_for_Scala/links/6213549a6c472329dcfa8c18/Region-based-off-heap-memory-for-Scala.pdf) indicates how `sun.misc.unsafe` can enable memory management in imem statically for JVM target.
