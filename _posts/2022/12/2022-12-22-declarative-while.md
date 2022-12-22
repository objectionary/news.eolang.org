---
layout: post
date: 2022-12-22
title: "The WHILE Object Is Declarative Now"
author: yegor256
---

In the recently released version [0.28.14](https://github.com/objectionary/eo/releases/tag/0.28.14)
we've changed the iterating algorithm of the
[`bool.while`](https://github.com/objectionary/home/blob/0.28.14/objects/org/eolang/bool.eo#L51-L56)
object. Until now,
by our mistake, it was imperative. Now, it's
[declarative](https://en.wikipedia.org/wiki/Declarative_programming).
The difference is in the result of its dataization.
The previous imperative version was "returning" a data object.
The new declarative one returns the latest body of the loop (without dataization!).
The difference is huge (thanks to it, many of our tests broke).

<!--more-->

This is how the iterating algorithm works now:

```
while($, ^, x):
  var last := TRUE
  var i := 0
  loop:
    if (not dataized(^)) break;
    dataized(last);
    last := x(^ := $, α0 := i);
    i := i + 1;
  return last;
```

Here, `^` is the parent of the `while` object, dataization of which returns a boolean value;
`$` is the `while` object itself;
and `x` is the object that is encapsulated
by the `while` object --- the _body_ of the [loop](https://en.wikipedia.org/wiki/For_loop).

Consider this simple loop:

```
memory -1 > x
while. > w
  x.lt 1
  [i]
    x.write i > @
```

It is equivalent to the following:

```
memory 0 > x
seq > w
  x.lt 1       # TRUE (x=-1)
  x.lt 1       # TRUE (x=-1)
  x.write 0
  x.lt 1       # TRUE (x=0)
  x.write 1
  x.lt 1       # FALSE (x=1)
  x.write 2
```

This may look counter-intuitive, but only because you may be used to imperative
loops in Java or Python, where variables are mutable and evaluations are eager.
In EO we have the opposite paradigm: variables are immutable and, more importantly,
evaluations are [lazy](https://en.wikipedia.org/wiki/Lazy_evaluation).

The `while` object first dataizes the body of the loop only when the result
of dataization may be ignored. However, the result of the last dataization in the loop
is important because it is what the `while` object "is" --- it is the body
of the loop after all pre-exit dataizations. In the example above, the `x.write 2` is
what the `while` object is (and the `seq` object too). Since `memory.write` is
what it writes into memory, the following holds:

```
w.eq 2
```

How is it possible to rewrite this code in order to make its flow of dataizations
be the following (a traditional imperative expectation, which is almost what we've had in EO before
the recent changes):

```
memory -1 > x
seq > w
  x.lt 1       # TRUE (x=-1)
  x.write 0
  x.lt 1       # TRUE (x=0)
  x.write 1
  x.lt 1       # FALSE (x=1)
  nop
```

The following code would work:

```
memory -1 > x
memory 0 > i
while. > w
  []
    if. > @
      x.lt 1
      seq
        x.write i
        i.write (i.plus 1)
        TRUE
      FALSE
  nop
```

Here, we abuse the design of the `while` object: the body is `nop`, while the
entire algorithm of the body and the condition are placed together into the
condition. Don't do this. But it works.
