# Plugin implementation

The following section provides an overview of the Scinear implementation.
For additional details, refer to the [Scinear Plugin repository](https://github.com/amsen20/scinear).

The Scinear implementation consists of two passes.
The first pass follows the [linear type system](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) typing rules and enforces linearity rules.
The second pass applies rules that are specific to Scinear.

# Linearity Rule Pass

## Traversing AST

The following section presents most of the AST nodes that Scinear supports and specifies the order in which Scinear traverses their children.
This order is a partial order, which means that, for some nodes, there is no specific order among their children.  
Applying this to the AST of a program produces a directed acyclic graph (DAG).
This DAG represents the traversal order that Scinear uses to enforce the linearity rule.

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

## The Pass Overview

The first pass traverses the AST using the `checkNode` function in [traversal order](#traversing-ast).
This traversal follows the typing rules described in the [main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf).

The recursive `checkNode` function maintains an assumption list when it reaches an AST node during traversal.  
This assumption list contains the linear variables that are accessible at that node.
The `checkNode` function then recursively visits the children of the node in traversal order and ensures that no linear value is used more than once.
In the case of children with no order between them, the intersection of their returned assumption lists is the resulting assumption list.
If the AST node is an expression, the function removes the used linear variables from the assumption list and returns the remaining list.
If the AST node is a block, a function definition, or a type definition, the `checkNode` function verifies that the remaining assumption list is empty.
This requirement ensures that every linear value is used exactly once.
Consider the following example:

```Scala
def f(x: LinearInt, y: LinearInt): LinearInt =
  //                ^ Linear argument y is not used
  if x.value > 0 then
    x // Linear value x is being used twice, or is not accessed directly
  else
    x // Linear value x is being used twice, or is not accessed directly
```

Scinear invokes `checkNode` on the body of the function definition with the assumption list containing `x` and `y`.
The block contains a single `if` expression. Therefore, the `if` node is checked recursively with the same assumption list, `x` and `y`.

The traversal order of the `if` expression is as follows:  

- First, the condition node `x.value > 0` is checked. After this step, the remaining assumption list is `y`, because `x` is consumed in the condition.  
- Next, the `then` and `else` branches are checked, in no specific order. Both branches reference `x`. However, `x` is no longer in the assumption list, which results in an error.

Finally, when control returns to the block node, `y` is still present in the assumption list. Consequently, Scinear reports an error indicating that `y` is not used.

# Scinear Specific Rules Pass

In this pass, Scinear checks the following properties:

- As discussed in the [internal and external fields section](./defining-linear-types.md#internal-and-external-fields), the linear fields of a linear class are divided into two disjoint sets: internal fields and external fields.
  Internal fields are not used outside the class definition, and external fields are not used within the class definition.
- As discussed in the [polymorphism section](./polymorphism.md#polymorphic-function-calls), polymorphic type parameters that are instantiated with linear types are either annotated with `@HideLinearity` or have a linear upper bound.

This pass is implemented as a conventional pattern-matching traversal.
