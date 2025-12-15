# Using Linear Types

Scinear follows the same [rule](../background/linear-types.md#main-linearity-rule) as [the main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf).
That is, a program accesses a linear value through a single reference, and this reference is read only once during execution.

To enforce this rule statically in a Scala program, Scinear tracks identifiers bound to values of linear types, called linear variables.

***Linear variable***:
A linear variable is an identifier that can be a method parameter, a class parameter, or a local variable that is bound to a value whose type is linear.

When an expression mentions a linear variable, the expression is using the variable.
The following rules limit the usage of linear variables in a program.

***Scinear-usage-at-most-rule:*** 
For each linear variable $l$ and any node $u$ that uses $l$, no node $v$ preceding $u$ in the [AST traversal order](../background/scala-ast.md#traversing-ast) may use $l$.

To put it another way, after a linear variable is used, it expires.
Keep in mind that the AST traversal order is a partial order.

***Scinear-usage-at-least-rule:*** 
The program has to use every linear variable statically.
To put it another way, for every linear variable, some expression in the program should mention it.

```Scala
class LinearInt(val value: Int) extends Linear

@main def main(): Unit =
  val x = LinearInt(1)
  val y = LinearInt(2)
  val z = // error haven't used `z` at all
    LinearInt(x.value + y.value) // cannot use `x` or `y` again after this point
end main
```

## Control flow

Usage rules influence the validity of mentioning linear variables within control-flow structures.
The following sections explain how Scinear treats control flow structures.

### If-Else Expressions

```Scala
if <condition expression> then <then expression> else <else expression>
```
Scinear first traverses the `condition` expression.
This means that the `then` expression, the `else` expression, and the rest of the program cannot refer to linear variables mentioned in the `condition`.
Because there is no traversal order between the `then` and `else` expressions, both expressions can refer to the same linear variables.
However, the plugin marks a linear variable as used for the remainder of the program if the variable appears in at least one branch.

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

```Scala
<match expression> match
  case <case pattern> => <case expression>
  ...
```
Scinear starts with the `match` expression.
The cases and the rest of the program cannot refer to the linear variables mentioned in this expression.
For each case, the traversal order is first the `case pattern` and then the `case expression`, meaning that the `case expression` should use linear variables defined in the `case pattern`.
Because there is no traversal order among cases, the plugin marks a linear variable as used for the remainder of the program if the variable appears in at least one of the `case pattern` or `case expression`.

<span id="linear-list-addToHead"></span>
```Scala
trait LinearList extends Linear
class Cons(val head: Int, val tail: LinearList) extends LinearList
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

### Try Catch Finally Expressions

```Scala
try
  <try expression>
catch
  case <case pattern> => <case expression>
  ...
finally
  <finally expression>
```
Scinear traverses the `try` expression first.
The catch `case` expressions, the `finally` expression, and the rest of the program cannot use linear variables mentioned in the try expression.
Similar to `match-case`, for each catch case, the traversal order is first the `case pattern` and then the `case expression`.
Because there is no traversal order between catch cases, these cases can use the same linear variables.
Scinear traverses the `finally` expression last.
As a result, the `finally` expression can only use linear variables that remain unused in the `try` expression and in all catch `case pattern`s and `case expression`s.

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

The loop body and condition are evaluated multiple times during execution.
Consequently, if a loop body or condition uses a linear variable, the program reads that variable multiple times during execution.

For example, the following demonstrates an incorrect implementation of a length function for a linear linked list.
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
In every iteration of the loop, the program reads the `curr` variable.
The correct way to implement the `length function` is using recursion, which is described in the next section.

To address this issue, Scinear enforces a specific rule for loops.

***Scinear-usage-loop-rule:***
The loop condition and body cannot use linear variables defined in the loop's enclosing scopes.

Recursion offers an alternative approach to this restriction. 
A recursive function safely consumes a linear value multiple times at runtime.

## Method and Function Definitions and Calls

Methods and functions present a similar problem to loops.
If a method or function body uses a linear variable, the program accesses the linear value via that variable multiple times during execution.

For example:
```Scala
def add(x: LinearInt, y: LinearInt): LinearInt =
  def addByXDef(z: LinearInt): LinearInt = 
    LinearInt(x.value + z.value) // error: `x` cannot be accessed here
  
  val addByXFunc = (z: LinearInt) => 
    LinearInt(x.value + z.value) // error: `x` cannot be accessed here
  
  LinearInt(x.value + y.value) // ok
end add
```
Every call to `addByXDef` or `addByXFunc` would result in reading linear variable `x`.

To resolve this issue, Scinear enforces a specific rule for anonymous functions and all definitions, including local, class member, and top-level package member methods.

<a name="scinear-usage-function-rule"></a>

***Scinear-usage-function-rule:***
The body of a method or a function cannot refer to a non-local linear variable.
In other words, the body can only refer to the linear variables defined as parameters or defined in the body itself.

The [main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) addresses this issue by treating the function itself as a linear value.
However, Scinear avoids this complexity by enforcing [the rule above](#scinear-usage-function-rule), at the expense of expressiveness.

The recursion approach to list length would look like this:
```Scala
def length(lst: LinearList): Int = lst match
  case Cons(head, tail) => 1 + length(tail)
  case nil: Nil => 0
end length
```

## Fields in other types

Following the [original linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf), Scinear does not allow classes that are not linear to have linear fields.

<a name="scinear-usage-nonlinear-type-field-rule"></a>
***Scinear-usage-nonlinear-type-field-rule:*** Nonlinear classes should not have linear fields.

```Scala
class Buffer(val list: LinearList) // error: nonlinear classes are not allowed to store linear fields
```
