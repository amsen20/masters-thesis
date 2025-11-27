# Polymorphism

Standard Scala 3 codebases use different aspects of polymorphism. Scinear integrates linear types into this polymorphism through two rules.

## Polymorphic promotion

The Scala standard library contains a rich set of types. Redefining linear versions for the entire library proves impractical. Scinear defines the following to address this limitation:

***Scinear-polymorphic-promotion-rule:*** Scinear assumes a polymorphic nonlinear type is linear if it uses a linear type as a type parameter.

The primary motivation for this rule is to prevent polymorphic types from holding linear fields without becoming linear themselves. However, the internal implementation of promoted types may violate Scinear rules.

This rule presents a trade-off between practicality and soundness. To preserve soundness, Scinear enforces a rule that limits polymorphic promotion's scope.

***Scinear-polymorphic-promotion-limitation-rule:*** Scinear only promotes `Option` and `Tuple` types. Other nonlinear types can have a type parameter instantiated with linear types if they are annotated with `@HideLinearity`.

The [`addToHead` example](/docs/scinear/using-linear-types.md#linear-list-addToHead) requires implementing an `unapply` method to actually work.
```Scala
object Cons:
  def unapply(l: Cons): Option[(Int, LinearList)] =
    Some((l.head, l.tail)) // Scinear exempts `unapply` method body from any linear rule.
end Cons
```
The following example presents an alternative implementation of `addToHead` for `Cons`.
```Scala
def addToHead(cons: Cons, offset: LinearInt): Cons =
  val deconstructedOpt = Cons.unapply(cons) // promoted `Option`
  val deconstructed = deconstructedOpt.get // promoted `Tuple`
  val (head, tail) = deconstructed
  val updatedHead = head + offset.value
  Cons(updatedHead, tail)
end addToHead
```

Linear types require the ability to decompose into fields to be practical. Scala implements this decomposition by returning an `Option[Tuple[...]]` of fields during `match-case` and `unapply` operations. This means that support for these two types is inevitable. Additionally, their simplicity ensures they are safe to promote.

## Polymorphic function calls

Polymorphic functions prevent code duplication. Furthermore, they are common in the Scala standard library. Calling these functions with linear values instantiates type parameters with linear types. Scinear follows a specific rule to track this instantiation.

***Scinear-polymorphic-function-call-rule:*** During a function call, if type parameter $T$ is instantiated with a linear type, three cases can happen:

1. $T$ is statically known to be linear: Scinear continues with no errors or warnings.  
2. $T$ is annotated with `@HideLinearity` annotation: Scinear emits a warning and continues.  
3. Otherwise, Scinear emits an error.

In the first case, Scinear validates the function body with the knowledge that $T$ is linear.
The second case is a way for developers to implement functions that do not follow linearity rules while still having linear parameters.
The last case prevents unintentional leakage of linear values through polymorphic interfaces.

```Scala
class LinearInt(val value: Int) extends Linear

def LinearIdentity[T <: Linear](x: T): T = // case 1
	// println(x) <-- error: would result in using `x` twice
	x
end LinearIdentity

def logIdentity[@HideLinearity T](x: T): T = // case 2
	println(x)
	x
end logIdentity

def normalIdentity[T](x: T): T = x // case 3

def checkIdentity(): Unit =
	LinearIdentity(LinearInt(42)) // ok
	logIdentity(LinearInt(42)) // ok
	normalIdentity(LinearInt(42)) // error: cannot pass linear value as polymorphic parameter
end checkIdentity
```
