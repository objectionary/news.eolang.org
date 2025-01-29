---
layout: post
date: 2023-09-04
title: "MEMORY as a container for a real data"
author: maxonfjvipon
---

In the recently released version [0.31.0](https://github.com/objectionary/eo/releases/tag/0.31.0),
we've changed the behavior of the `memory` object. Until now, when we stored an object in `memory`,
it attempted to dataize the object into data and started behaving accordingly. However, we are now
reevaluating the concept of data within our language, and the `memory` is an entry point.

<!--more-->
Let's delve into the topic of data for a moment. In every programming language, we are accustomed to
representing data using various primitive types, or in our case, objects such as `int`, `float`,
`bytes`, `string`, and `bool`. But upon closer examination, you may observe that four of them can
be reduced to `bytes`, making it unnecessary to single them out.

Henceforth, there are only two fundamental entities in our language: objects and data, with data
being merely a sequence of bytes, nothing more.

Now, let's revisit how we used `memory` previously. Consider the following code, which illustrates a
typical usage of `memory`:

```
memory 0 > mem

while. > @
  mem.lt 10
  [i]
    mem.write > @
      mem.plus 1
```
Here, after storing an `int` in `memory`, it behaves as if it were an `int`, allowing access to all
internal objects of the `int`. In this case, `int` acts as data.
However, from now on, an `int` is just a regular object, and `bytes` is the sole representation of data.
`memory` adheres to these rules. The following code demonstrates how the behavior of `memory` has been altered:

```
memory 0 > mem

while. > @
  mem.as-int.lt 10
  [i]
    mem.write > @
      mem.as-int.plus 1
```

Here, after storing an `int` in `memory`, it behaves like `bytes` – genuine data.

Additionally, it's important to remember that `memory` is not unlimited. Therefore, we have
introduced a new restriction: when `memory` accepts and converts an object into data for the first
time, it records the object's length in bytes. Subsequently, it is not permissible to write an
object whose length in bytes exceeds that of the original object.

As a result, the following code will fail:

```
memory TRUE > mem # <-- 1 byte is written here

mem.write 10 > @  # <-- 8 bytes are written here
```

However, it's crucial to note that `memory` only retains the length of the first stored object.
Consequently, the following code will NOT fail:

```
memory 8 > mem   # <-- 8 bytes are written here

seq > @
  mem.write TRUE # <-- 1 byte is written here
  mem.write 10   # <-- 8 bytes are written here again
```

That's all for today, be in touch!
