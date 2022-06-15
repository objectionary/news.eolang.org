---
layout: post
date: 2022-06-15
title: "Introduced `string.slice` Object"
author: Graur
---

Since 0.23.8 version it is possible to use
[`string.slice`](https://github.com/objectionary/home/blob/master/objects/org/eolang/string.eo)
object just like this:

```
"Привет".slice 1 2
```

Here, `slice` object returns "ри" string, which is a substring of "ри" string
from the specified `start` position and with the length `len`.

If above parameters will be out of bound, `slice` will return
an `error` object instead of a new `string` object.

<!--more-->

Visit our [paper](https://arxiv.org/abs/2206.02585) to get more details.
