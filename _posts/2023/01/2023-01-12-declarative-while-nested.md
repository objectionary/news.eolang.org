---
layout: post
date: 2023-01-12
title: "Declarative WHILE: nested example"
author: mximp
---

Previous [post](https://news.eolang.org/2022-12-22-declarative-while.html)
by @yegor256 explained declarative WHILE nature.
Let's examine more complex example when `while` objects are nested.

<!--more-->

Consider the following code:

```eo
+alias org.eolang.txt.sprintf

[] > nested-while
  memory 0 > x
  memory 0 > y
  seq > @
    while.
      x.lt 10
      [i]
        seq > @
          y.write 0
          while.
            y.lt 2
            [i]
              seq > @
                write.
                  y
                  y.plus 1
                write.
                  x
                  x.plus 1
    QQ.io.stdout
      sprintf
        "x=%d"
        x
```

Here, we have two `while` loops one inside the other. Initially `x` is set to `0`.
The outer loop continues while `x` lower than `10`. Inner loop executes its body
twice every time it's dataized.

So what do you think the `x` will be after dataization of the whole program?
The answer is `15` and here is why.

Every time the inner `while` is dataized, `x` is incremented by `3`: two times as
`while` iteration and one more time once the body is returned as a result. So the single
outer `while` iteration increments `x` by three. The last true condition of the outer
`while` would be after `3` iterations when `x=3*3=9`. The next iteration would set
`x` to `12` which will stop the outer loop.

After that, due to its declarative nature, the outer `while` will supply its body object, which
will be dataized one more time as requested by `seq` object. And the body happens to be the
inner `while`:

```eo
[i]
  seq > @
    y.write 0
    while.
      y.lt 2
      [i]
        seq > @
          write.
            y
            y.plus 1
          write.
            x
            x.plus 1
```

And we already know that this object would increase `x` by `3`. So the final value of `x` will
be set to `15`.

Sometimes it might be tricky to reason about the result when working with complex objects in EO.
But it's only a matter of practice and getting used to its declarative nature.
