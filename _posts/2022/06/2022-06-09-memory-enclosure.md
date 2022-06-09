---
layout: post
date: 2022-06-09
title: "Memory Can't Be Empty"
author: yegor256
---

Until version 0.23.7 it was possible to use
[`memory`](https://github.com/objectionary/home/blob/master/objects/org/eolang/memory.eo)
object just like this:

```
memory > m
m.write 42
```

At the first line, a copy of `memory` was made and then labeled as `m`. This
was a bug in the language. The object `memory` must not be copied if there
are no arguments provided for the copying (application) operation.

<!--more-->

The right way since 0.23.7 is this:

```
memory 0 > m
m.write 42
```

Here, the object `m` is a copy of `memory` with a single argument, which is
called an "enclosure".

We also deleted the attribute `memory.is-empty`, since `memory` is always
not empty.

The same changes were applied to the object
[`cage`](https://github.com/objectionary/home/blob/master/objects/org/eolang/gray/memory.eo).
