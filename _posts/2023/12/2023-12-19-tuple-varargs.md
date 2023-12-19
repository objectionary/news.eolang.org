---
layout: post
date: 2023-12-19
title: "Recursive tuple and varargs"
author: maxonfjvipon
---

We're continuing to observe new features of the latest release 
[0.34.1](https://github.com/objectionary/eo/releases/tag/0.34.1) of EO, and today we talk about new
recursive implementation of `tuple` object and why we got rid of varargs in our language.

<!--more-->
### Recursive tuple
As it was mentioned in the [previous](https://news.eolang.org/2023-12-08-phi-and-unphi-mojos.html) 
blog post, EO is based on φ-calculus, which is a formal model that we are attempting to use as 
a base for object-oriented programming languages. There are only two fundamental entities in the 
calculus - objects and data. Object is a collection of uniquely named attributes and data is just a
sequence of bytes.

There's also a special attribute `Δ`(delta) which is attached to the data. In our vision of EO only
one object should have such an attribute - `bytes`. But for a long time at least 6 objects had it: 
`bytes`, `int`, `float`, `string`, `bool` and (surprise) `tuple`. The `Δ` attribute of `tuple` 
stores array of java `Phi` objects, which was totally wrong.

But the decision was made - only `bytes` should have a `Δ` attribute, so we need to remove it from 
other 5 objects including `tuple`. And it was a good chance to rewrite `tuple` in pure EO which
allowed us to decrease amount of [atoms](https://news.eolang.org/2022-12-02-java-atoms.html).

So, welcome new implementation of `tuple` written in pure EO:
```
# Tuple.
[head tail] > tuple
  # Empty tuple
  [] > empty
    [i] > at
      error "Can't get an object from the empty tuple" > @
    [x] > with
      tuple > @
        tuple.empty
        x
    0 > length

  # Obtain the length of the tuple.
  [] > length
    ^.head.length.plus 1 > @

  # Take one element from the tuple, at the given position.
  [i] > at
    ^.length > len!
    if. > index!
      i.lt 0
      len.plus i
      i
    if. > @
      or.
        index.lt 0
        index.gte len
      error "Given index is out of tuple bounds"
      if.
        index.lt (len.plus -1)
        ^.head.at index
        ^.tail

  # Create a new tuple with this element added to the end of it.
  [x] > with
    tuple > @
      ^
      x
```

As you may see `tuple` has two free attributes now: `head` and `tail`; where the `head` is always 
supposed to be a `tuple` as well and `tail` is the object we want to store.

For example, if we want to create a tuple of one object it would look like this:
```
tuple > my-tuple
  tuple.empty
  1
```

The `tuple.empty` is a special `tuple` that stores nothing and used as a top level `tuple` in whole 
recursion.

If we want to create a tuple of three objects it would look like this:
```
tuple > my-tuple
  tuple
    tuple
      tuple.empty
      1
    2
  3
```

I hope you get the idea. We also left the `*`(star) syntax sugar which allows you to write a `tuple`
in one line:

```
* 1 2 3 > my-tuple
```

This example is equivalent to the previous one and will be transformed by compiler to it. 

### Varargs

Before the release [0.34.0](https://github.com/objectionary/eo/releases/tag/0.34.0) there were 
varargs in EO which allow you to have an object with uncertain amount of free attributes. For 
example `int.plus` object looked something like this:  

```
[] > int
  [x...] > plus
    # code here 

5.plus 5 10 > twenty
20.plus 1 2 3 4 > thirty 
```

It was quite convenient mechanism, but it didn't match to φ-calculus where every attribute has to 
be attached to the concrete object. With varargs application `5.plus 5 10` looks something like 
this:

```
Φ.org.eolang.int(Δ ⤍ 5).plus(
  α0 ↦ Φ.org.eolang.int(Δ ⤍ 5),
  α1 ↦ Φ.org.eolang.int(Δ ⤍ 10)
)
```

But there aren't two free attributes in formation of `int.plus`, only one `x`:

```
Φ ↦ [org ↦ [eolang ↦ [int ↦ [plus ↦ [x ↦ ∅, ...], ...], ...], ...], ...]
```

So as not to complicate things we decided to get rid of varargs in EO at all. And now it's a perfect
match between EO and φ-calculus. Of course readability suffered a little because of it, but it 
worth that.

Next time we'll put an end to data primitives, be in touch.