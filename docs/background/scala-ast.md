# Scala Abstract Syntax Tree

The Scala compiler, similar to other compilers, processes a program through several [phases](https://www.scala-lang.org/api/3.4.2/docs/docs/contributing/architecture/phases.html).
The program starts as text and eventually becomes executable/interpretable code for the target platform.
Each phase receives the program, modifies, annotates, or validates it, and then produces an updated version.
The input and output of each phase are intermediate representations (IRs) of the program.  

In Scala, these IRs are represented as Abstract Syntax Trees (ASTs) until the final backend phases.
Scala [plugins](https://docs.scala-lang.org/scala3/reference/changed-features/compiler-plugins.html) can access and process the AST at any phase.
The AST becomes typed after the typer phase and retains Scala type information until the erasure phase.

Scinear is a compiler plugin that processes the AST after the `cc` phase, which performs capture checking and annotates types with capture information, and before the `erasure` phase.

## AST Nodes

The following is the list of AST nodes that Scinear supports:

- **`tpd.ValDef`**: A value definition, such as `val name = rhs`.
- **`tpd.DefDef`**: A method or function definition, such as `def name(param1, ..., paramN) = rhs`.
- **`tpd.Block`**: A block expression `{ stat1; ...; statN; expr }`.
- **`tpd.If`**: A conditional expression `if cond then thenp else elsep`.
- **`tpd.Match`**: A pattern-match expression `selector match { case1 ... caseN }`.
- **`tpd.CaseDef`**: A single case clause `case pat if guard => body` inside a match expression.
- **`tpd.Try`**: A try-catch-finally expression `try block catch { case1 ... caseN } finally finalizer`.
- **`tpd.Assign`**: A variable assignment `x = rhs`.
- **`tpd.WhileDo`**: A while loop `while cond do body`.
- **`tpd.Select`**: A field or method selection `qualifier.member`.
- **`tpd.Typed`**: A type ascription `expr: T`, where the expression is explicitly annotated with a type.
- **`tpd.SeqLiteral`**: A sequence literal `[elem1, ..., elemN]`.
- **`tpd.NamedArg`**: A named argument in a function call, such as `name(x = arg)`.
- **`tpd.Annotated`**: An annotated expression `arg: @annotation`.
- **`tpd.Return`**: A return statement `return expr`.
- **`tpd.Labeled`**: A labeled expression used to represent named blocks that can be exited early.
- **`tpd.Inlined(expansion)`**: An inlined expression that results from macro expansion, where `expansion` is the inlined body.
- **`tpd.TypeDef`**: A type or class definition, such as `class name { stat1; ...; statN }`.
- **`Util.Call`**: A function or method call `ref(arg1, ..., argN)`, where `ref` is the function reference and the arguments are flattened.
- **`Util.NewExpr`**: An object instantiation `new TypeName(arg1, ..., argN)`.
- **`Util.PolyFun`**: A polymorphic function literal, such as `[TypeNameT] => rhs`.
- **`tpd.ClosureDef`**: A closure or anonymous function definition, such as `name => rhs`.
