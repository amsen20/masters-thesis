# Capture checking

Scala includes a new experimental feature called capture checking. [Based this section](/docs/background/capturing-types.md), a capture set consists of a set of terms and the universal capability (`cap`). A type includes a capture set in two locations:

* As a type parameter: `T[CS^={...}]`  
* The type’s capturing set: `T^{...}`

***Scinear-capture-set-rule:*** Let AST node $u$ be an expression of type $T$. Assume $T$ has a capture set type parameter $CS$ that mentions a linear term $l$. No node $v$ that precedes $u$ in traversal order may mention $l$.

This rule disallows mentioning a linear term in a capture set after the linear term has expired.

```Scala
TODO AN EXAMPLE WITH CAPTURE SET PARAMETERS
```

Keep in mind that this rule only tracks capture sets in type parameters. However, capturing sets may still include an expired linear term. The `imem`’s use case motivates this design choice. 

In general, the developer has less control over capturing sets than over the capture set type parameters. The capturing set can resolve to `{cap}` at any program point. This resolution is more restrictive than the original set. However, it excludes the linear term.

```Scala
TODO AN EXAMPLE WITH TYPE CAPTURING SET.
```
