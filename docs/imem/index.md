# Static memory management library `imem`

In this chapter, we introduce `imem`, a library for Scala that provides static (compile-time) memory management. At first, we explore how we want our memory model to behave during runtime, and what rules are applied to it. Then, the chapter explains how ownership is implemented using linearity and how we can utilize capture checking for borrow checking. At the end, the details of `imem` integration into Scala native are explained.
