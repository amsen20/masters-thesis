# Scinear: A linear type plugin for Scala

Scinear is a [compiler plugin](https://docs.scala-lang.org/scala3/reference/changed-features/compiler-plugins.html) for the Scala 3 language. It introduces [linear types](/docs/background/linear-types.md) to standard Scala code and strictly enforces their usage rules. This enforcement mechanism provides the minimal feature set necessary to implement the [`imem`](../imem/index.md) library. Consequently, the plugin prioritizes [`imem`](../imem/index.md) requirements rather than serving as a general-purpose linearity tool.
