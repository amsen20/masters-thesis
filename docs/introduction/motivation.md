# Motivation

This example is a minimal version of the web-crawlers benchmark presented in [this](https://github.com/amsen20/web-crawlers-bench/tree/main) project, which exposed the garbage-collection performance problem in Scala Native to me.
The following program implements a simple, single-threaded web crawler in Scala:

```Scala
def crawl(toCrawlLinks: Queue[String], shouldStop: Boolean): Unit =
  if shouldStop then toCrawlLinks.toList.foreach(println(_))
  else
    val (currentLink, q) = toCrawlLinks.dequeue
    val newLinks = process(httpGet(currentLink))
    println(currentLink)
    crawl(q.enqueueAll(newLinks), checkShouldStop())
```

This function is tail-recursive and takes two arguments.
The first argument is the queue of links to crawl, and the second argument indicates whether execution should stop.
The `crawl` function first checks the stop condition.
If execution should stop, the function prints all remaining links in the queue.
Otherwise, it removes a link from the queue, extracts the links from the web content of that link, prints the current link, and then recursively calls `crawl`.

The `toCrawlLink` queue and its underlying linked list grow as the program progresses.
The Scala Native garbage collector scans the entire linked list at each GC pause.
Because the linked list continues to grow, the duration of each pause also increases over time.
This behavior leads to longer pauses and a reduction in Scala Native performance.

One way to prevent these pauses is to use [Scala Zones](https://github.com/scala-native/scala-native/pull/3120) and allocate the queue inside a zone.
In this case, when the zone goes out of scope, the queue is automatically freed.
However, this approach leads to a memory leak.
The popped links, which the program prints, are no longer needed, but they remain allocated in the zone, and they are not freed individually and remain in memory until the entire zone, and therefore the whole queue, is released.

The following demonstrates a simplified version of the same program that uses imem:

```Scala
def crawl[OQ^, ...](toCrawlLinks: Box[Queue[String, OQ^], OQ], shouldStop: Boolean)(...): Unit =
  if shouldStop then
    foreach(toCrawlLinks, println(_))
  else
    val toCrawlLinks2 =
      val lf = Lifetime[...]()
      val (queueMutRef, queueBoxHolder) = borrowMutBox[...](toCrawlLinks)
      val (queueMutRef2, newLinks) =
        val lf2 = Lifetime[...]()
        val (queueRefForDequeue, queueMutRefHolder) = borrowMut[...](queueMutRef)

        val currentLinkBox: Box[String, {lf2, ...}] = popQueue(queueRefForDequeue)
        val (currentLink, _) = borrowImmutBox[...](currentLinkBox)
        println(currentLink)
        val newLinks = process(httpGet(currentLink))

        (unlockHolder(lf2.getKey(), queueMutRefHolder), newLinks)

      enqueueAll(queueMutRef2, newLinks)
      unlockHolder(lf.getKey(), queueBoxHolder)

    crawl(toCrawlLinks2, checkShouldStop())
```

The implementation details of what imem interfaces are and how to use them are explained thoroughly in the [imem](../imem/index.md) and [evaluation](../imem/index.md) chapters.
The following explains the example briefly.

The program accesses the queue through a `Box`.
[Boxes](../imem/ownership.md), similar to [Rust](../background/rust.md)’s boxes, are references to which imem controls program access statically.
imem enforces static ownership and borrowing rules for accessing a box.
The `Box` and the `Queue` data structure both take `OQ^` as a type parameter.
`OQ^` is similar to Rust’s lifetime annotations.
It indicates that both the box and the queue to which it points remain valid as long as all lifetime capabilities in the set that instantiates `OQ^` are available.

The function first checks whether it should stop.
If execution should stop, the function uses `foreach` to print all links that remain in the queue.

If execution should not stop, the function defines a new lifetime `lf`.
This is the lifetime of the mutable reference that the program borrows to access the queue through `toCrawlLinks`.
imem does not allow direct access to a box referent, which is called a resource.
Instead, the program has to borrow the box, either immutably or mutably, and then access the box resource through the borrowed references.

After borrowing `toCrawlLinks`, the program loses access to `toCrawlLinks` because `Box` is a linear type and its instances follow the [linearity rule](../background/linear-types.md).
The `borrowMutBox` function returns a mutable reference to the queue, `queueMutRef`, and a value holder, `queueBoxHolder`.
The `queueBoxHolder` holds the queue until the program unlocks it using the key of lifetime `lf`.
When the program unlocks the holder using `unlockHolder(lf.getKey(), queueBoxHolder)`, the lifetime `lf` expires.
Then, the program can no longer mention the mutable reference and all other objects that include `lf` in their lifetime set, and they can be freed.
This behavior enables imem to give temporary access to the box resource through mutable or immutable references, and then return the box while also invalidating the borrowed references.

After borrowing `toCrawlLinks`, the program re-borrows as `queueRefForDequeue` because `queueMutRef` is linear and the function needs access twice.
One use is for popping from the queue, and the other use is for pushing to the queue.
Similarly, the `borrowMut` function returns a value holder, `queueMutRefHolder`, which holds the mutable reference `queueMutRef` as long as `lf2` is available.

The function pops `currentLinkBox` from the queue and then borrows it immutably to print the link and process the web content of that link.
After that, the function unlocks `queueMutRefHolder` to get `queueMutRef2` back.
The program enqueues all new links using `enqueueAll(queueMutRef2, newLinks)`, and then unlocks `queueBoxHolder` to get the queue box back.
Finally, the function uses that box to recursively call `crawl`.

Regarding memory management:

- The queue lifetime is `OQ^`, which is determined by the function caller and known to the program statically.
  Therefore, there is no need for a garbage collector to watch this value and determine when it is possible to free the object.

- The `currentLinkBox`, which is the popped link, has a lifetime set that includes `lf2`.
  Therefore, when `lf2` expires at `(unlockHolder(lf2.getKey(), queueMutRefHolder), newLinks)`, this box and its resource are ready to be freed, and the program can no longer access the box or its resource.
  This behavior resolves the memory leakage problem.

A final note is that the current implementation of imem does not manage memory allocations.
Instead, it only applies ownership and borrowing rules to enforce safe memory management, alias control, and mutability control statically.
The [future works](../conclusion/future-works.md) section sketches how runtime memory management can be supported in the future.
