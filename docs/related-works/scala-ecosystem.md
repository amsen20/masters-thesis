# Surrounding Projects in Scala

This section focuses on attempts in the Scala ecosystem that target static and safe memory management.

## Scala Safe Zones

Scala [SafeZone](https://github.com/scala-native/scala-native/pull/3120)s implement region-based, or zone-based, static memory management in Scala.
They operate in Scala Native and integrate well with the garbage collector.

In brief, zones are lexical scopes that provide [escape-checked](../background/capturing-types.md#escape-checking) access to an instance of `SafeZone`.
When the program allocates a new object, it can pass the zone instance so that the object is allocated within that zone.
When the scope of the zone ends, the zone and all objects allocated in it are freed.  
Moreover, the allocation result captures the zone instance and is therefore also bound to the zone’s scope, making access to the object statically safe.

Safe zones address a simplified version of the problem that imem targets.
They do not model borrowing or ownership.
All aliases to an object have the same mutability access, and they share the same lifetime.
In addition, objects cannot be moved from one zone to another without copying them.

Safe zones use a `nativelib` memory pool to manage zone and object allocations at runtime, which could serve as inspiration for imem in native Scala memory management.

## Region-based Off-heap Memory For Scala

[This report](https://www.researchgate.net/profile/Denys-Shabalin/publication/291105865_Region-based_off-heap_memory_for_Scala/links/6213549a6c472329dcfa8c18/Region-based-off-heap-memory-for-Scala.pdf) implements a [cyclone](./cyc-to-rust.md#cyclone) style region-based memory management in Scala targeting JVM.

Three versions of region based memory management are implemented:

- **Unchecked:**
This is the most performant version that doesn't do any static or dynamic checks to ensure that no dangling pointers exist.
- **Dynamic:**
  In this version, the memory accesses are dynamically checked to make sure that the allocated page exists.
- **Static:**
  Using literal-based singleton types, each region is assigned a unique type and using macros all field accesses and instances of the off-heap classes are ensured to have an implicit instance of that region.
  In this way statically there cannot be any dangling references although the program has to open and close the regions.

Similar to Safe zones, compared to imem, this research targets a simpler problem.
This approach does model ownership and borrowing.
As a result, there is no static alias management and mutability control.

A JVM paginated memory allocation runtime is implemented to support this approach that demonstrates that the unchecked version can perform as good as the GC version but with half memory usage.
imem can also use this runtime to perform memory management in JVM.

[This report](https://www.researchgate.net/profile/Denys-Shabalin/publication/291105865_Region-based_off-heap_memory_for_Scala/links/6213549a6c472329dcfa8c18/Region-based-off-heap-memory-for-Scala.pdf) presents a [Cyclone-style]([cyclone](./cyc-to-rust.md#cyclone)) region-based memory management approach for Scala on the JVM.

It implements three versions of region-based memory management:

- **Unchecked:**
  The most performant version, with no static or dynamic checks to prevent dangling pointers.

- **Dynamic:**
  Performs runtime checks on memory accesses to ensure the addressed page exists.

- **Static:**
  Uses literal-based singleton types to assign a unique type to each region.
  With macros, all field accesses of off-heap classes require an implicit instance of the corresponding region.
  This guarantees, at compile time, that no dangling references exist, although regions must still be explicitly opened and closed.

Similar to Safe Zones, and compared to imem, this work addresses a simpler problem.
It does not model ownership or borrowing, and therefore does not provide static alias management or mutability control.

The implementation includes a JVM-based paginated memory allocation runtime.
The results show that the unchecked version can match GC performance while using roughly half the memory.
This runtime could also be used by imem to manage memory on the JVM.

## System Capybara

[System Capybara](https://2025.workshop.scala-lang.org/details/scala-2025/6/System-Capybara-Capture-Tracking-for-Ownership-and-Borrowing) is an ongoing project that extends capture checking to support ownership and borrowing, enabling static alias and mutability control.

The extension ensures that fresh, non-consuming capabilities are mutually exclusive within the capture set in which they are instantiated.
For consuming arguments, it guarantees that after a function call, the program cannot refer to anything that captures the consumed capabilities.
The system also introduces read-only references whose capture sets may safely share capabilities.

Capybara addresses the same overall goal as imem, which is Rust-style ownership and borrowing, but takes a fundamentally different approach.
The goal of imem is to rely primarily on existing features of the Scala type system, capture checking, and dependent types, and add well-studied linear types, and combine them with capture checking.
In contrast, Capybara extends capture checking directly to support ownership and borrowing.

As a result, Capybara does not explore the potential of combining two distinct systems, capture checking and linearity, to achieve ownership and alias control, features that neither system fully targets on its own.
