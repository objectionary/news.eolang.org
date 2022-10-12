---
layout: post
date: 2022-07-18
title: "Introduced 'error' Object and new semantic of 'try' Object"
author: Graur
---

Since 0.25.0 version it is possible to use
[`error`](https://github.com/objectionary/home/blob/master/objects/org/eolang/error.eo)
object just like this:

```
[x] > check
  if. > @
    x.eq 0
    error "Can't divide by zero"
    42.div x
```

Here, the object `error` causes program termination at the first attempt to dataize it.
It encapsulates any other object, which can play the role of an exception that is
floating to the upper level.

The object `try` enables the catching of an `error` objects and extracting exceptions from them.
For example, the following code prints "The 1th argument of 'int.div' is invalid: division by zero is infinity" and then "finally":

```
QQ.try
  []
    42.div 0 > @
  [e]
    QQ.io.stdout > @
      e
  []
    QQ.io.stdout > @
      "finally"
```
The object `error` encapsulates the error message from `int.div` object and terminates the program.

<!--more-->

We can also throw the `error` through multiple nested `try` objects:

```
eq. > @
  QQ.try
    []
      QQ.try > @
        []
          slice. > @
            "some string"
            7
            5
        [e]
          e > @
        []
          "first finally block" > @
    [e]
      e > @
    []
      "second finally block" > @
  "Start index + length must not exceed string length but was 12 > 11"
```

Here, we get `TRUE` because `string.slice` object uses `error` object.
Which encapsulates the error message "Start index + length must not exceed string length but was 12 > 11".
The first `try` object throws up this encapsulated string to the second `try` object.

In addition, you can add new behavior to catch errors, or just use them as is.
For example, the following code prints "division by zero is infinity, please check the parameter"

```
QQ.try
  []
    3.div 0 > @
  [e]
    QQ.io.stdout > @
      e.slice 43 28
  []
    QQ.io.stdout > @
      ", please check the parameter"
```

Visit our [paper](https://arxiv.org/abs/2206.02585) to get more details.
