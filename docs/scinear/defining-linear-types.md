# Defining Linear Classes

Defining a linear class is the same as defining a standard class in Scala.
A class, and in general a type, is linear if it extends the `scinear.Linear` trait.
This extension informs the scinear compiler plugin to enforce linearity rules on the class.
The following demonstrates a basic declaration:

```Scala
import scinear.Linear

class LinearInt(val value: Int) extends Linear
```

[The original linear types paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) views linear types as immutable algebraic data types.
Scinear adopts this view by enforcing specific rules on field and method definitions within a linear class.

## Field Definition Rules

### Internal and External Fields

In Scala, a class field may reference other fields within the same class during initialization.
However, this access violates the [linearity rule](../background/linear-types.md#main-linearity-rule) during the construction of linear classes.
For example, consider the following definitions:

```Scala
class LinearDouble(val value: Double) extends Linear

class Circle(val r: LinearDouble) extends Linear:
  val circumference = LinearDouble(2 * math.Pi * r.value)
end Circle

@main def main(): Unit =
  val circle = Circle(LinearDouble(5.0))
  // circle.r <-- error accessing an internal member
  circle.circumference // ok external member access
end main
```

If Scinear permits accessing `r` outside the class body, such as in the `main` function, `circle.r` is read twice,
once during the construction of `circle` and again in the `main` function.

***Scinear-fields-rule-1:***
Every linear field of a linear class should be either an internal field or an external field.
These categories are defined as follows:

- **Internal fields** that other fields refer to in the class body (i.e., during the execution of the constructor).  
- **External fields** that are referred to from outside the class instance (i.e., through a selector).

This rule manages access scope.
During initialization, it allows some fields' initial value expressions to refer to an internal linear field.
After construction, it permits access to an external linear field through a selector (e.g., `linearInstance.linearField`).
The rule mutually excludes internal and external fields, meaning the program does not read a linear field of a linear instance more than once.

Scinear analyzes the class body, starting with the class parameters, then proceeding through the statements in program order.
The [usage section](./using-linear-types.md) explains the order's effect.

### Fields immutability

The program reads each field of a linear instance only once, so defining mutable fields in a linear class is unnecessary.

***Scinear-fields-rule-2:***
All fields of a linear class should be strictly evaluated immutable values (val), not mutable variables (var) or lazy values (lazy val).
Each field’s type can be either linear or nonlinear.

This rule enforces immutability on linear types and simplifies the evaluation order.

```Scala
class LinearList(val next: LinearList) extends Linear:
  // var size: Int = 1 + next.size <-- error variable field

  // lazy val size: Int = 1 + next.size <-- error lazy val field

  val size: Int = 1 + next.size // ok
end LinearList
```

## Method Definition Rules

### Methods and Linear Fields

When a class member method references a field in its body, every invocation of the method potentially reads the field.
This access causes a problem if the field is linear.
For example, in the following:
```Scala
class LinearList(val next: LinearList) extends Linear:
  def getSize: Int = 1 + next.getSize // error `next` is not accessible here
end LinearList
```
Every call to `getSize` would result in reading the `next` field.

***Scinear-method-rule1:***
A class member's methods defined within a linear class cannot refer to the class's linear field.

This rule prevents the linear fields of a class from leaking into its methods. 
Furthermore, this restriction ensures that linear classes behave similarly to a datatype.

### Exempted Methods

Inherited and compiler-generated methods are inevitable.
Also, the developer cannot modify the implementation of these methods.
Moreover, linear classes are useful only if they can be decomposed into their fields.
In Scala, this operation requires the class to implement the `unapply` method.
However, following Scinear rules, implementing an `unapply` method is impossible.
For example:
```Scala
class LinearPair(val x: LinearInt, val y: LinearInt) extends Linear
object LinearPair:
	def unapply(p: LinearPair): (LinearInt, LinearInt) = (p.x, p.y)
end LinearPair
```
It is not possible to implement the `unapply` method without referring to `x` and `y` fields.

***Scinear-method-rule2:***
In a linear class, inherited methods from Object, compiler-generated methods, and the `unapply` method defined in the linear class companion object are exempt from all linear rules.

These method exemptions maintain practicality within a standard Scala 3 codebase.

## Complementary Rules

### `this` in Linear Classes

Accessing `this` implies reading the value of the instance referred to by the keyword.
This reference would violate the [linearity rule](../background/linear-types.md#main-linearity-rule) if the instance is linear.
For example:
```Scala
class LinearList(val data: Int, val next: LinearList) extends Linear:
  def append(newData: Int): LinearList =
    LinearList(newData, this) // error `this` is not allowed to refer to linear values
end LinearList
```
Every call of the `append` method on a `LinearList` instance would result in reading the instance.

***Scinear-self-reference-rule:***
A program must not use the keyword `this` in any scope where it refers to an instance of a linear class.

This rule ensures that a linear class's fields cannot refer to the linear instance during the instance's construction. 
Thus, the instance is always available for use after construction.
Furthermore, a linear class's method body does not refer to the class instance during a method call, meaning that after reading a linear instance to get access to its method, the linear instance is not reconsumed by calling its method.

### Type Hierarchy Separation

***Scinear-inheritance-rule:*** A linear class can only extend linear types and `Object`.

This rule separates linear type hierarchy from other types, joining them at `Object`.

### Class Definition Simplification

***Scinear-Nesting-rule:*** Linear class definitions cannot include nested class definitions.

This rule reduces complexity of defining linear classes without sacrificing much practicality.
