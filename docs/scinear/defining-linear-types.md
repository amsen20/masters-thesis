# Defining Linear Types

Defining a linear type is the same as defining a standard type in Scala. A type is linear if it extends the scinear.Linear trait. This extension informs the scinear compiler plugin to enforce linearity rules on the class. The following demonstrates a basic declaration:

```Scala
import scinear.Linear

class LinearInt(val value: Int) extends Linear
```

[The original linear types paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf) views linear types as immutable algebraic data types. Scinear adopts this foundational definition by enforcing specific rules on field and method definitions within a linear type.

## Field Definition Rules

***Scinear-fields-rule-1:*** the plugin divides the type’s linear fields into two disjoint subsets:

* **Internal fields** that other fields can use during the construction phase.  
* **External fields** that can be used only from outside the class instance.

This rule manages access scope. It allows some linear fields to use other linear fields during initialization and to access some fields from outside, while preventing any linear field from being used more than once.

The construction phase starts with the class parameters and then proceeds through the statements in program order. The [usage section](./using-linear-types.md) explains the order's effect.

```Scala
class LinearDouble(val value: Double) extends Linear

class LinearCircle(val r: LinearDouble) extends Linear:
  val circumference = LinearDouble(2 * math.Pi * r.value)
end LinearCircle

@main def main(): Unit =
  val circle = LinearCircle(LinearDouble(5.0))
  // circle.r <-- error accessing an internal member
  circle.circumference // ok external member access
end main
```

***Scinear-fields-rule-2:*** All fields should be strictly evaluated values (val), not variables (var) or lazy values (lazy val). Each field’s type can be either linear or non-linear.

This rule enforces immutability on linear types and simplifies the evaluation order.

```Scala
class LinearList(val next: LinearList) extends Linear:
  // var size: Int = 1 + next.size <-- error variable field

  // lazy val size: Int = 1 + next.size <-- error lazy val field

  val size: Int = 1 + next.size // ok
end LinearList
```

## Method Definition Rules

***Scinear-method-rule:*** Linear type’s methods cannot reference linear fields inside their bodies. In other words, linear fields are excluded from the method’s closure environment. Methods that are inherited from Object, compiler-generated methods, and `unapply` method are exempt from this rule.

This rule prevents linear fields from leaking into methods. This restriction ensures the type functions similarly to a datatype. Concurrently, the method exemptions maintain practicality within a standard Scala 3 codebase.

```Scala
class LinearList(val next: LinearList) extends Linear:
  def getSize: Int = 1 + next.getSize // error `next` is not accessible here
end LinearList
```

## Complementary Rules

***Scinear-self-reference-rule:*** `this` keyword should never refer to a linear type.

By following this rule, the initialization process does not consume the instance. Thus, after creating a linear type instance, the creator can use it. Furthermore, a method invocation does not reuse the instance.

```Scala
class LinearList(val data: Int, val next: LinearList) extends Linear:
  def append(newData: Int): LinearList =
    LinearList(newData, this) // error `this` is not allowed to refer to linear values
end LinearList
```

***Scinear-inheritance-rule:*** A linear type can only extend other linear types or `Object`.

Because of this rule, linear types will always remain linear or `Object` through type casting.

***Scinear-Nesting-rule:*** Linear type definitions cannot include nested class definitions.

This rule reduces linear type complexity without sacrificing much practicality.