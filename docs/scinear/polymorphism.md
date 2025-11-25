# Polymorphism

Standard Scala 3 codebases use different aspects of polymorphism. Scinear integrates linear types into this polymorphism through two rules.

## Polymorphic promotion

The Scala standard library contains a rich set of types. Redefining linear versions for the entire library proves impractical. Scinear defines the following to address this limitation:

***Scinear-polymorphic-promotion-rule:*** Scinear assumes a polymorphic nonlinear type is linear if it uses a linear type as a type parameter.

```Scala
TODO AN EXAMPLE WITH EXPLANATION
```

The primary motivation for this rule is to prevent polymorphic types from holding linear fields without becoming linear themselves. However, the internal implementation of promoted types may violate Scinear rules.

This rule presents a trade-off between practicality and soundness. To preserve soundness, Scinear enforces a restrictive rule.

***Scinear-polymorphic-promotion-limitation-rule:*** Scinear only promotes `Option` and `Tuple` types. Other nonlinear types can have a type parameter instantiated with linear types if they are annotated with `@HideLinearity`.

Linear types require the ability to decompose into fields to be practical. Scala implements this decomposition by returning an `Option[Tuple[...]]` of fields during `match-case` and `unapply` operations. This means that support for these two types is inevitable. Additionally, their simplicity ensures they are safe to promote.

## Polymorphic function calls

Polymorphic functions prevent code duplication. Furthermore, they are common in the Scala standard library. Calling these functions with linear values instantiates type parameters with linear types. Scinear follows a specific rule to track this instantiation.

***Scinear-polymorphic-function-call-rule:*** During a function call, if type parameter $T$ is instantiated with a linear type, three cases can happen:

1. $T$ is statically known to be linear: Scinear continues with no errors or warnings.  
2. $T$ is annotated with `@HideLinearity` annotation: Scinear emits a warning and continues.  
3. Otherwise, Scinear emits an error.

In the first case, Scinear validates the function body with the knowledge that $T$ is linear. 

```Scala
TODO AN EXAMPLE OF FIRST CASE
```

The second case is a way for developers to implement functions that do not follow linearity rules while still having linear parameters.

```Scala
TODO AN EXAMPLE OF THE SECOND CASE
```

The last case prevents unintentional leakage of linear values through polymorphic interfaces.

```Scala
TODO AN EXAMPLE OF THE THIRD CASE
```
