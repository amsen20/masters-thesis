# Capture checking

Scala includes a new experimental feature called capture checking.
[Based on this section](/docs/background/capturing-types.md), a capture set consists of a set of capabilities.
A capture set appears in a type in two locations:

* As a capturing type's capture set annotation: `T^{/* capture set */}`
* As a type argument instantiating a type parameter with a capture set: `T[{/* capture set */}]`.

***Linear capability:***
A linear capability is a capability whose type is linear.
In other words, the capability's type extends the `scinear.Linear` trait.

***Scinear-capture-set-rule:***
Let AST node $u$ be an expression of type $T$.
Assume $T$ has a type parameter $CS >: caps.CapSet$ that the argument instantiating it includes a linear capability $l$.
$l$ should not be expired in node $u$ meaning no node $v$ that precedes $u$ in traversal order may refer to $l$.

This rule disallows mentioning a linear reference, which is also a capability, in a capture set after the program has used it.

```Scala
class Tracker[LinearCaps^]:
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
In this example, `Tracker` is a nonlinear type that tracks linear capabilities within its type parameter `LinearCaps`.
The program can refer to `tracker` only if all linear capabilities in the `LinearCaps` type argument, `{x, y}` in the example above, are not expired.

Keep in mind that this rule only tracks capture sets that instantiate a type parameter.
However, capture sets annotating capturing types may still include an expired linear capabilities.
The `imem`’s use case motivates this design choice.

In general, capture sets annotating capturing types are covariant.
This covariance means that they can be widened to `{cap}` through avoidance at any point in the program.
The widened capture set, `{cap}`, is more restrictive than the original set.
However, it excludes the linear capability.

The following example uses an `Object`'s capture set to track `{x, y}`.
Scinear enforces no limitations on `tracker` in this case.
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
