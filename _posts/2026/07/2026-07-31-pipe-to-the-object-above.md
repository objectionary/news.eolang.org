---
layout: post
date: 2026-07-31
title: "Pipe to the Object Above"
author: yegor256
---

In [𝜑-calculus][phi], a formation may be applied on the spot:
  the expression `⟦a ↦ 7, x ↦ ∅⟧(x ↦ 42)` forms an object
  with one void attribute and immediately supplies the argument.
EO had no way to say this.
A formation first had to receive a name,
  and then a separate application had to repeat that name.
[Recently][issue] we closed the gap with a new syntax element:
  a line that starts with `|`, which we call a *pipe*.

<!--more-->

### One Character, Pointing Up

A pipe line applies arguments to the object declared just above it,
  at the same indentation:

```
[x] > foo
  x.plus x > @
| 5 > foo5
```

The formation `foo` doubles its argument;
  the pipe applies `5` to it and names the result `foo5`.
The `|` reads as an arrow pointing up:
  "take the object above, give it `5`, call the result `foo5`."
The semantics is untouched —
  the pipe desugars into a plain application,
  and the program above compiles to exactly the same XMIR as:

```
[x] > foo
  x.plus x > @
foo 5 > foo5
```

Sugar, pure sugar.
So why bother?

### Lambdas, Finally Callable

Because the name may not exist.
The `>>` suffix [auto-names][auto] an abstract object
  when a meaningful name would add nothing,
  and the generated name is deliberately untypeable.
Until now, such an object could only be passed somewhere as an argument;
  it could not be applied, since application requires a name to repeat.
The pipe needs no name:

```
[a b] >>
  a.plus b > @
| 2 3 > pair
```

This is a lambda expression applied at the place of its definition —
  what the [original issue][issue] asked for:
  anonymous formations working as functions in Haskell
  or lambdas in Java.

### Chains, Verticals, Dots

Consecutive pipes stack into left-associated applications:

```
[x] > foo
  x > @
| 1 >>
| 2 > result
```

Here `result` is `(foo 1) 2`.
Each pipe in a chain is an object of its own
  and must carry a name,
  because the next pipe refers to its predecessor by it —
  `>>` does the job when you don't care.

When arguments don't fit on one line,
  the pipe takes them vertically,
  exactly as any application head does:

```
[x y] > add
  x.plus y > @
| > sum
  2
  3
```

And since a pipe application is a complete value,
  a `.method` may follow it:

```
[x] > foo
  x > @
| 5
.neg > y
```

### The Rules

The pipe is picky about its predecessor.
It must be a formation or another pipe,
  and it must be named —
  either explicitly with `> name` or automatically with `>>`:

```
6 > six
| 5 > n
```

This is rejected: `six` is a number, not a formation.
A pipe with nothing above it is rejected too,
  and so is a pipe after a `.method` line:
  once the dot has dispatched an attribute,
  the formation is no longer the object in hand.

### What It Buys

The [runtime library][eo] already uses pipes,
  mostly for one pattern:
  a local recursive helper, defined and called in one breath.
This is how [`number.power`][power] computes an integer exponent:

```
[b n] >> ipow
  if. > @
    n.eq 0
    1
    if.
      is-integer. (n.div 2)
      half.times half
      b.times
        ipow b (n.minus 1)
    ipow b (n.div 2) > half
| base (abs expo)
```

The helper stays anonymous to the enclosing scope,
  yet recursion works:
  `>> ipow` gives the formation a file-local handle,
  the body calls itself by it,
  and the pipe fires the first call with the actual arguments.
Definition and invocation sit together;
  the reader never hunts across the file
  for the place where `ipow` is used.

In 𝜑-calculus, formation-with-application was there all along.
Now EO can write it too,
  without inventing a name for the middle.

That's all for today.

[phi]: https://arxiv.org/abs/2111.13384
[issue]: https://github.com/objectionary/eo/issues/5432
[auto]: https://news.eolang.org/2025-02-21-auto-named-abstract-objects.html
[power]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/number/power.eo
[eo]: https://github.com/objectionary/eo
