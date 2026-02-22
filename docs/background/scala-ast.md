# Scala Abstract Syntax Tree

The Scala compiler, similar to other compilers, processes a program through several [phases](https://www.scala-lang.org/api/3.4.2/docs/docs/contributing/architecture/phases.html).
The program starts as text and eventually becomes executable/interpretable code for the target platform.
Each phase receives the program, modifies, annotates, or validates it, and then produces an updated version.
The input and output of each phase are intermediate representations (IRs) of the program.  

In Scala, these IRs are represented as Abstract Syntax Trees (ASTs) until the final backend phases.
Scala [plugins](https://docs.scala-lang.org/scala3/reference/changed-features/compiler-plugins.html) can access and process the AST at any phase.
The AST becomes typed after the typer phase and retains Scala type information until the erasure phase.

Scinear is a compiler plugin that processes the AST after the `cc` phase, which performs capture checking and annotates types with capture information, and before the `erasure` phase.

## Traversing AST

The following presents the list of all AST nodes and the order in which their children are traversed in the context of Scinear.
This order is a partial order, which means that, for some nodes, there is no specific order among their children.  
Applying this to the AST of a program produces a directed acyclic graph (DAG).
This DAG represents the traversal order that Scinear uses to enforce the linearity rules.

Sequential Ordered Nodes:

- **`Util.Call`**: first `ref`, then arguments (`arg1`, ..., `argN`) in argument order after flattening.
- **`Util.NewExpr`**: arguments (`arg1`, ..., `argN`) in argument order after flattening.
- **`tpd.Block`**: first statements (`stat1`, ..., `statN`) in program order, then `expr`.
- **`tpd.SeqLiteral`**: elements (`elem1`, ..., `elemN`) in program order.
- **`tpd.CaseDef`**: first `pat`, then `guard`, then `body`.
- **`tpd.TypeDef` (linear)**: template statements (`stat1`, ..., `statN`) in program order.

Branching Nodes:

- **`tpd.If`**: first `cond`, then `thenp` and `elsep`, no order between `thenp` and `elsep`.
- **`tpd.Match`**: first `selector`, then cases, `case1`, ..., `caseN`, no order among cases.
- **`tpd.TypeDef` (non-linear)**: template statements, `stat1`, ..., `statN`, independently, no order among them.
- **`tpd.Try`**: first `block`, then cases, `case1`, ..., `caseN`, no order among cases, then `finalizer` after `block` and all `cases`.

The following AST nodes each have a single child; therefore, the traversal order is trivial:

- `tpd.Select`: `qualifier`
- `tpd.Typed`: `expr`
- `tpd.NamedArg`: `arg`
- `tpd.Assign`: `rhs`
- `tpd.Annotated`: `arg`
- `tpd.Return`: `expr`
- `tpd.Labeled`: `expr`
- `tpd.Inlined`: `expansion`
- `tpd.ValDef`: `rhs`

The following nodes isolate some of their children.
Therefore, these children appear as independent sources in the ordering DAG:  

- `tpd.WhileDo`: the `cond` and `body` children are isolated; therefore, both are sources of the DAG.  
- `tpd.DefDef`, `tpd.ClosureDef`, `Util.PolyFun`: the `rhs` child is isolated and thus a source of the DAG.
