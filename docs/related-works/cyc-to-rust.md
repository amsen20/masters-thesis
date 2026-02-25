# Cyclone, Vault and Rust

This section describes the key features, the main approach, and examples of three chosen well-known projects that target static memory management and static mutability control.
These projects are discussed in chronological order.

## Cyclone

[Cyclone](https://dl.acm.org/doi/abs/10.1145/512529.512563) is a type-safe programming language that is based on C.
Cyclone provides region-based static memory management through its static type system.
The memory of a Cyclone program is divided into regions.
These regions are of three kinds:

- A single heap region that lives forever.
- Stack regions that correspond to stack frames containing local declarations.
- Dynamic regions that the program defines, whose lifetime is bounded by lexical scope.

Each value in memory resides in a region, and every pointer type is annotated with the region to which the pointer refers.

Cyclone allows functions and structures to be region-polymorphic.
Therefore, the system is expressive and duplication is reduced.
In addition, for expressiveness, Cyclone supports sub-typing.
A pointer to one region can be used as a pointer to a different region if the pointer’s region outlives the target region.

Cyclone prevents dangling pointers by disallowing the dereferencing of pointers whose associated region is no longer live.
To prevent buffer overflow, Cyclone provides fat pointers that augment pointers with runtime bounds information.
For static mutability, it enforces stricter rules than C on `const` pointers within its type system.
Furthermore, to reduce manual region annotations, Cyclone performs pattern-based implicit region annotation.

To provide an overview of Cyclone, the following shows the `longest` function implemented in Cyclone:

```C
char?`r longest<`r>(const char?`r s1, const char?`r s2) {
    if (strlen(s1) > strlen(s2)) {
        return s1;
    } else {
        return s2;
    }
}
```

The function takes two pointers to strings and returns a pointer to the longer string.
The pointers are fat pointers to a character, which are annotated with the region `r`, that is, `const char?'r`.

This function is memory safe because all arguments and the returned pointer are annotated with the same region.
Therefore, the region of the return type must be a subtype of the regions of all arguments.
This requirement means that both argument regions must outlive the region of the returned pointer.

## Vault

[Vault](https://dl.acm.org/doi/10.1145/512529.512532) is an experimental programming language that extends the [linear type system](../background/linear-types.md) to address the restrictive aliasing rules of linear programs.
In linear programs, only one reference to a linear value is allowed, and any structure that contains a linear field must itself be linear.
This restriction is limiting and contrasts with many common programming patterns, such as sharing handles to a storage resource where one handle at a time has write access, while all handles have read access.
To address this issue, Vault introduces non-linear, shareable, immutable views of a linear value after its *adoption*.
Furthermore, during *focus*, one of these non-linear views is granted limited linear, mutable access to the underlying linear value.

Vault introduces the following concepts:

- **Key:** A key represents the static name of the memory address of an object.  
- **Capability:** A capability defines a mapping between a key and its current type.  

The compiler maintains a set of capabilities for each scope.

After adoption, the program transfers ownership of a linear value, called the *adoptee*, to another linear value, called the *adopter*.
After this transfer, the program no longer has linear access to the adoptee, and the adoptee's key is removed from the capabilities.
Instead, the program has non-linear access to the adoptee through a reference with a guarded type.
This guarded type informs the compiler that the reference can access the value only if the *adopter* key is present in the capabilities.

During focus, which is a scoped operation, the program gains linear access through a guarded reference.
Within this scope, the key of the guarded reference is removed from the set of capabilities.
As a result, other guarded references with the same key cannot obtain focus views within that scope.

## Rust

[Rust](https://dl.acm.org/doi/abs/10.1145/3158154), which is explained in detail in [backgrounds](../background/rust.md), is inspired by Cyclone, Vault, and many others.
It provides static memory management and static mutability control.
Rust enforces static memory management by applying ownership rules.
Moreover, it achieves memory safety and static mutability control through borrow checking.
In addition, Rust performs fully automated region inference aided by lifetime annotations.
When automatic pattern-based lifetime inference is insufficient, it allows manual lifetime annotations to guide the analysis.

## imem's Standpoint

imem is primarily inspired by Rust.
In particular, the concepts of lifetimes, boxes, and immutable and mutable access are borrowed from Rust.

imem follows rules that are similar to Rust in many places. However, the main difference is that imem is not a new programming language.
Instead, imem is a library, and it does not require dedicated language support to provide static, safe memory management and mutability control.

imem relies on existing features of the Scala type system, including capture checking and dependent types.
In addition, it uses a well-known foundation, linear types, which Scinear, a minimal compiler plugin, provides.

Overall, imem shows that static, safe memory management can be achieved by combining existing type system features, instead of designing new compiler features and introducing additional complexity to the type system from scratch for this purpose.

