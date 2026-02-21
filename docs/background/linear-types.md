# Linear types

The paper *Linear Types Can Change the World* proposes linear types to address a problem in the way functional programming languages model updates to the *world*.
Here, the *world* refers to the large program state around which functionality is built, for example, the file system in a program that performs many file-system operations.
In functional programming languages, the program can duplicate or discard this state at any point.
However, both actions may lead to performance and correctness problems.  

To address this problem, program memory is divided into two parts:

- **Linear memory:**
  All accesses to this part of memory follow the linearity rule. That is, values of linear types have exactly one reference.

- **Nonlinear memory:**
  This part of memory has no restrictions, except that a value of nonlinear type cannot reference a value of linear type.

As a result, the linear memory forms a collection of trees.
Each linear value has exactly one reference.
For the roots of these trees, the references are variables in the program environment.
The linear memory may reference the nonlinear memory, but not vice versa.

The paper extends a typed version of the lambda calculus by adding linear types to the type system.
Moreover, it statically ensures that the linearity rule is always satisfied.
To achieve this property, each expression is associated with an assumption list that contains all free variables that have a linear type that the expression may reference.
The type system then guarantees that each expression consumes all variables in its assumption list.
In addition, the list is divided into disjoint parts, which ensures that each variable is used exactly once.
Moreover, the expression cannot reference any linear variable other than those listed in its assumption list.

For example, consider the following Scala code:  

```Scala
def f(x: LinearInt, y: LinearInt): Unit =
  println(x)
  println(y)
```  

The expression `println(x); println(y)` has the assumption list `x` and `y`.
The expression `println(x)` has the assumption list `x`.
Also, the expression `println(y)` has the assumption list `y`.

As a result, the following program is rejected by the linear type system:  

```Scala
def f(x: LinearInt): LinearInt =
  println(x)
  x
```  

The expression `println(x); x` has the assumption list `x`.
However, both the sub-expression `println(x)` and the sub-expression `x` require `x` in their assumption lists.
Therefore, one of them would have to omit `x`, which violates the linearity rule.
Consequently, the program is invalid.
