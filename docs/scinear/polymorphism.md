# Polymorphism

Standard Scala 3 codebases use different aspects of polymorphism.
Scinear integrates linear types into this polymorphism through two rules.

## Polymorphic promotion

Instantiating a polymorphic type with a linear type argument results in a nonlinear value that stores a linear value, as below:
```Scala
val ls: List[LinearInt] = List(LinearInt(42))
ls(0) // accessing to a linear value
ls(0) // accessing to a linear value
```
This outcome contradicts [the main linearity paper](https://www.researchgate.net/profile/Philip-Wadler/publication/2429119_Linear_Types_Can_Change_the_World/links/6410b420315dfb4cce7cf9bc/Linear-Types-Can-Change-the-World.pdf)'s rule that nonlinear values cannot retain linear values.
Also, Scinear enforces a [specific rule](./using-linear-types.md#scinear-usage-nonlinear-type-field-rule) to prevent nonlinear classes from storing linear fields.
But this rule is limited to class definitions.
Scinear addresses linear instantiations through linear promotion.

***Linear promotion***:
If Scinear promotes a nonlinear type, the plugin enforces all linearity rules on that instances of that type.
Put another way, Scinear treats promoted types as linear types.

One approach is to promote all linear instantiations of polymorphic types.
However, this strategy is unsound because the internal implementation of promoted types may violate Scinear rules.

Furthermore, banning all linear instantiations is impractical because linear types require the ability to decompose into fields.
Scala implements this decomposition by returning an `Option[Tuple[...]]` of fields during `match-case` and `unapply` operations.
This means that supporting linear type arguments for these two polymorphic types is inevitable.
As an example, the [`addToHead` example](./using-linear-types.md#linear-list-addToHead) requires implementing an `unapply` method to actually work:
```Scala
object Cons:
  def unapply(l: Cons): Option[(Int, LinearList)] =
    Some((l.head, l.tail)) // Scinear exempts `unapply` method body from any linear rule.
end Cons
```
Moreover, the simplicity of `Option[T]` and `Tuple[T]` ensures that promoting these types is safe.

To address all the discussed issues safely, Scinear enforces a minimal promotion rule.

***Scinear-polymorphic-promotion-rule:***
If a method parameter, class parameter, local variable, or a `this` reference `v` possesses type $T[T_1, ..., T_n]$ where $T$ is not a linear type and type parameter $T_i$ is instantiated with a linear type, one of the following cases applies:

1. If $T$ is `Option` or `Tuple`, Scinear promotes $T$ to a linear type and treats `v` as a linear reference.
2. If $T_i$ has the `@HideLinearity` annotation, Scinear emits a warning.
3. Otherwise, Scinear emits an error.

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

## Polymorphic function calls

Polymorphic functions prevent code duplication.
Plus, they are common in the Scala standard library.
Calling these functions with linear values instantiates type parameters with linear types.
Scinear follows a specific rule to track this instantiation.

***Scinear-polymorphic-function-call-rule:*** During a function call, if type parameter $T$ is instantiated with a linear type, three cases can happen:

1. $T$ has a linear upper bound: Scinear continues with no errors or warnings.
2. $T$ is annotated with `@HideLinearity` annotation: Scinear emits a warning.
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
