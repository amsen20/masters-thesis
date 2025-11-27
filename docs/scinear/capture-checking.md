# Capture checking

Scala includes a new experimental feature called capture checking. [Based this section](/docs/background/capturing-types.md), a capture set consists of a set of terms and the universal capability (`cap`). A type includes a capture set in two locations:

* As a type parameter: `T[CS^={...}]`  
* The type’s capturing set: `T^{...}`

***Scinear-capture-set-rule:*** Let AST node $u$ be an expression of type $T$. Assume $T$ has a capture set type parameter $CS$ that mentions a linear term $l$. No node $v$ that precedes $u$ in traversal order may mention $l$.

This rule disallows mentioning a linear term in a capture set after the linear term has expired.

```Scala
class Tracker[LinearTerms^]:
	def check(): Unit = ()
end Tracker

@main
def main(): Unit =
	val x: LinearInt^ = LinearInt(10)
	val y: LinearInt^ = LinearInt(20)
	val tracker: Tracker[{x, y}] = Tracker()
	tracker.check()
	println(x.value)
	// tracker.check() <-- error: `tracker` mentions `x` which is expired in its capture set.
	println(y.value)
end main
```
In this example, `Tracker` is a nonlinear type that tracks linear terms within its capture set parameter. 
The program can refer to `tracker` only if all terms in its capture set, `{x, y}`, exist and are not expired.

Keep in mind that this rule only tracks capture sets in type parameters. However, capturing sets may still include an expired linear term. The `imem`’s use case motivates this design choice. 

In general, the developer has less control over capturing sets than over the capture set type parameters.
The capturing set can resolve to `{cap}` at any program point. This resolution is more restrictive than the original set. However, it excludes the linear term.

The following example uses an `Object`'s capture set to track `{x, y}`. Scinear enforces no limitations on `tracker` in this case.
Also, this example demonstrates that the tracking list is easily eliminated through avoidance.
```Scala
@main
def main(): Unit =
	val x: LinearInt^ = LinearInt(10)
	val y: LinearInt^ = LinearInt(20)
	val tracker: Object^{x, y} = Object()
	val avoidedTracker: Object^ = tracker // does not mention `x` or `y` in its type.
	tracker // ok
	println(x.value)
	tracker // ok
	println(y.value)
end main
```
