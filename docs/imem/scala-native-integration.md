# Scala native integration

Until now, we discussed how the `imem` enforces ownership rules using the Scala type system and `Scinear`. This section focuses on how `imem` can allocate memory, keep it out of the garbage collector's reach, and free it when the owner object is out of scope.

## Overview of GC internals

## Manually managed references

## Wrapping unsafe references inside `imem` references
