# Using Linear Types

Scinear enforces usage rules throughout the program to correctly track terms with linear type usages. These constraints ensure that the program mentions linear terms exactly once.

***Scinear-usage-at-most-rule:*** For each term $t$ with a static linear type in a [Scala program AST](/docs/background/scala-ast.md). For any usage node $u$ of term $t$, no other usage node $u'$ of $t$ precedes $u$ in the AST traversal order.

Keep in mind that the AST traversal order defined [here](/docs/background/scala-ast.md#traversing-ast) is a partial order. 

***Scinear-usage-at-least-rule:*** Each term $t$ with a static linear type in a Scala program AST is at least used once.

```Scala
class LinearInt(val value: Int) extends Linear

@main def main(): Unit =
  val x = LinearInt(1)
  val y = LinearInt(2)
  val z = // error haven't used `z` at all
    LinearInt(x.value + y.value) // cannot use `x`` or `y` after this point
end main
```

## Control flow

Usage rules influence the validity of linear terms within control-flow structures. The following sections explain how Scinear treats control flow structures.

### If-Else Expressions

Scinear first traverses the condition expression. This means that the then expression, the else expression, and the rest of the program cannot refer to linear terms mentioned in the condition. Because there is no traversal order between the then and else expressions, both expressions can refer to the same linear terms. However, the plugin marks a linear term as used for the remainder of the program if the term appears in at least one branch.

```Scala
class LinearInt(val value: Int) extends Linear

def select(a: LinearInt, b: LinearInt, c: LinearInt, d: LinearInt): LinearInt =
  if (a.value > b.value)
    // cannot use `a` and `b` here
    d
  else
    // cannot use `a` and `b` here
    c
  // cannot use `a`, `b`, `c`, and `d` here
end select
```

### Case-Match Expressions

Scinear starts with the `match` expression. This means that the cases and the rest of the program cannot refer to the linear terms mentioned in this expression. Because there is no traversal order among cases, the plugin marks a linear term as used for the remainder of the program if the term appears in at least one `case` expression.

```Scala
trait LinearList extends Linear
class Cons(val head: Int, val tail: LinearList) extends LinearList
object Cons:
  def unapply(l: Cons): Option[(Int, LinearList)] =
    Some((l.head, l.tail))
end Cons
class Nil() extends LinearList

def addToHead(lst: LinearList, offset: LinearInt): LinearList = lst match
  case Cons(head, tail) =>
    // cannot use `lst` here
    Cons(head + offset.value, tail)
  case nil: Nil =>
    // cannot use `lst` here
    nil
  // cannot use `lst` and `offset` here
end addToHead
```

### Try-Catch-Finally Expressions

Scinear traverses the try expression first. This means that the catch clauses, the finally block, and the rest of the program cannot refer to linear terms mentioned in the try block. Because there is no traversal order between catch clauses, these clauses can refer to the same linear terms. Scinear traverses the finally expression last. Consequently, this block can only use linear terms that remain unused in the try block or in any catch clause.

```Scala
def tryF(f: LinearInt => LinearInt, x: LinearInt, default: LinearInt): LinearInt =
  try
    val y = LinearInt(x.value + 1) // local to this block
    f(y)
  catch
    case _ =>
      // cannot use `f` and `x` here
      default
  // cannot use `f`, `x` and `default` here
end tryF
```

### Loops

Scinear enforces a new rule on loops to ensure linearity.

***Scinear-usage-loop-rule:*** The loop condition and body expressions cannot refer to any linear term available in their environment.

This rule prevents a loop from consuming linear values multiple times at runtime.

```Scala
def length(lst: LinearList): Int =
  var curr = lst
  var len = 0

  while !curr.isInstanceOf[Nil] do // error cannot use `curr` in loop's condition
    curr = curr match // error cannot use `curr` in loop's body
      case Cons(_, tail) =>
        tail
      case nil: Nil =>
        nil
    len += 1
  end while
  
  len
end length
```
This example demonstrates an incorrect implementation of a length function for a linear linked list. The correct approach is using recursion, which next section describes.

Recursion offers an alternative approach to this restriction. A recursive function safely consumes a linear value multiple times at runtime. The following section describes this alternative.

Function definition and calls

Scinear views functions as more generalised forms of the methods. So, it enforces a specific rule for them.

***Scinear-usage-function-rule: F***unction bodies, and in general all closures, can only use linear terms in their parameters.

If a linear term is captured in a closure, multiple evaluations of the closure expression will result in multiple uses of the linear value. The [main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) addresses this by treating the closure as a linear value itself. But in Scinear, this rule avoids complexity by sacrificing expressiveness.

## Function Definitions and Calls

Scinear views functions as generalized methods. Consequently, the plugin enforces a specific rule for these structures.

<a name="scinear-usage-function-rule"></a>

***Scinear-usage-function-rule:*** Function bodies and closures can only use linear terms present in their parameter list.

```Scala
def add(x: LinearInt, y: LinearInt): LinearInt =
  def addByXDef(z: LinearInt): LinearInt = 
    LinearInt(x.value + z.value) // error: `x` cannot be accessed here
  
  val addByXClosure = (z: LinearInt) => 
    LinearInt(x.value + z.value) // error: `x` cannot be accessed here
  
  LinearInt(x.value + y.value) // ok
end add
```

Capturing a linear term in a closure poses a problem. Multiple evaluations of the closure expression consume the linear value multiple times. The [main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) addresses this issue by treating the closure itself as a linear value. However, Scinear avoids this complexity by enforcing [the rule above](#scinear-usage-function-rule), at the expense of expressiveness.

```Scala
def length(lst: LinearList): Int = lst match
  case Cons(head, tail) => 1 + length(tail)
  case nil: Nil => 0
end length
```
This example demonstrates the recursion approach to list length.

## Fields in other types

Following the [original linearity paper]((https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf)), Scinear does not allow other types to have linear fields.

***Scinear-usage-nonlinear-type-field-rule:*** Nonlinear types should not have linear fields.

```Scala
class Buffer(val list: LinearList) // error: Non-linear types are not allowed to store linear values
```
