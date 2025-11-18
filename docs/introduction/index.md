# Introduction

Starts with explaining Scala being a garbage collected language on JVM and Native platforms and then showcasing the problem with GC pauses using a Scala program (probably the simplified version of the program that will be used in the benchmarks, for now let's consider it's a web crawler program that the main list of all crawled links is the main cause of the GC pauses). Then, it mentions that the list has to be around the whole program execution, and tracing it is a waste of time that grows with the list's length. In the end, it uses the proposed library to manage the list memory management statically.

FOCUS ON A SIMPLE PEACE OF THE WEB CRAWLERS
