---
layout: post
date: 2022-08-31
title: "New dataization result of While object"
author: Graur
---

As in all other programming languages, an object [while](https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/org/eolang/bool.eo#L55) 
is used to iterate over a set of statements while a condition is true. 
In [EO](https://github.com/objectionary/eo) this object not only does this, 
but is also dataized. And the result of such dataization was the number of iterations
performed. Now this behavior has changed: the result of such dataization can now 
be any object (depending on the condition).
It can be divided into two parts: when the condition is `TRUE` and when the condition
is `FALSE`. 

When the condition is `TRUE` we get the result of the last dataization object in the `while` loop.

When the condition is `FALSE` such dataization result become `TRUE`.

<!--more-->

Let's check it out with examples. In this case, the result of condition `x.lt 6` 
is `TRUE` for at least one iteration. So the result of dataization `when-true` object is `6`:
```
[] > when-true
  memory 3 > x
  while.
    x.lt 6
    [i]
      seq > @
        x.write (x.plus i)
        x
```

And in this case, the result of condition `x.lt 6` is `FALSE` for at least one iteration.
So the result of dataization `when-false` object is `TRUE`:
```
[] > when-false
  memory 10 > x
  while.
    x.lt 6
    [i]
      seq > @
        x.write (x.plus i)
        x
```