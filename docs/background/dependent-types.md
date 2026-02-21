# Path Dependent Types

Scala provides dependent types through path-dependent types, which are formalized in [DOT](https://infoscience.epfl.ch/server/api/core/bitstreams/678b1818-06aa-47f1-b7e0-783413d76044/content) and in [*Foundations of Path-Dependent Types*](https://dl.acm.org/doi/epdf/10.1145/2714064.2660216).
The following briefly explains the features of path-dependent types that are relevant to imem.

In Scala, classes can declare type members in addition to methods and immutable or mutable fields.
For example, in the following code, class `A` defines a field `x`, a method `f`, and a type member `C`:

```Scala
class A:
  val x = 1
  def f(): Unit = ()
  class C()
end A
```

In Scala path-dependent types, a path typically begins with an immutable variable.
It is followed by zero or more selections of immutable fields and ends with a selection of a type member.
Each such path denotes a unique type.
For example, consider the following code:

```Scala
val a = A()
val b = A()

val _: a.C = a.C() // ok
val _: b.C = b.C() // ok
val _: a.C = b.C() // compile error:
//           ^^^^^
//           Found:    b.C
//           Required: a.C
```

In the example above, the program can instantiate `a.C` and `b.C`.
However, when it attempts to assign an instance of `b.C` to a variable of type `a.C`, a compile-time error occurs because these two types are distinct.

In imem, path-dependent types are used by defining a class that contains an `opaque` type member.
This usage is illustrated in the following example:

```Scala
class Lifetime:
  opaque type Key = Object

  def getKey(): Key = Object()
end Lifetime
```

In this way, the program can access an instance of the `Key` type associated with a given path only by using the same path to invoke the `getKey` method.
This behavior is illustrated below:

```Scala
val lf = Lifetime()
val _: lf.Key = lf.getKey() // ok
val _: lf.Key = lf.Key() // error
val _: lf.Key = Object() // error
val lf2 = Lifetime()
val _: lf.Key = lf2.getKey() // error
```
