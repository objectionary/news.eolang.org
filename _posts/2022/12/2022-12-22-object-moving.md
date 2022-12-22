---
layout: post
date: 2022-12-22
title: "Object Adoption"
author: yegor256
---

In the recently released version [0.28.14](https://github.com/objectionary/eo/releases/tag/0.28.14)
we introduced a new language feature, which is called "object adoption"
--- because it allows changing of an object's parent. Every object in
[EO](https://www.eolang.org) has a parent object, which is set to it when it's born
(either formed or copied). Until the recent release it was
not possible to change the parent object using EO language. However, it
was possible to do this from inside an atom (through Java).
Now, there is no special API inside Java, but the feature is available through EO.

<!--more-->

This is how an object is "formed":

```
[id] > book
  "Object Thinking" > title
```

This is how we make a "copy" of it:

```
book 42 > b
```

For both of them the parent is the root object `Q`. You can access it
using the `^` identifier:

```
QQ.io.stdout
  "The title is %s"
  b.^.book.title
```

Now, it's possible to create a new copy of `book` with a different parent:

```
book > x
  42
  b:^
```

Here, `^` (after the colon) is the name of the attribute to be bound to `b`.
The `x` object created will have `^` bound to `b`.

With the help of this object adoption mechanism, instead of this:

```
a.if "left" "right"
```

We can do this:

```
bool.if
  a:^
  "left"
  "right"
```

And, finally, the actual adoption happens when we take an closed object
and re-bind its `^` attribute:

```
b > y
  book:^
```

Here, we create a copy of `b`, name it `y`, and reset the parent to the `book` object.

The objects `goto`, `bool.while` and `try` use object adoption mechanism. For example,
when `while` dataizes the encapsulated body, it first adopts it:

```
[] > foo
  memory TRUE > x
  x.while
    [i]
      ^.^.write FALSE > @
```

Here, `^.^` points to the parent object of the parent object. The parent
of the abstract anonymous object is the `while` object. The parent of `while`
is `x`. However, it is not true until the dataization of the `while` starts.
When the program starts, the parent of the abstract anonymous object
is the `foo` object. It is the `while` object who adopts its copies after
creating them.

Thus, the `^` attribute is the only _mutable_ attribute in EO.

