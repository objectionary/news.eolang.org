---
layout: post
date: 2026-07-31
title: "The Terminator"
author: yegor256
---

Four years ago we [introduced][old] the `error` and `try` objects:
  `error` wrapped an object and threw it up the stack,
  `try` caught it, extracted the payload, and ran a `finally` block.
It worked, but it was Java smuggled into EO —
  [𝜑-calculus][phi] has no exceptions to throw and no stack to unwind.
What it does have is `⊥`, the *bottom* — a computation that terminated.
[Recently][issue] we deleted `try` and `error`
  and gave the bottom a spelling of its own:
  the letter `T`, which we call the *terminator*.

<!--more-->

### A Value That Detonates

The [new syntax element][syntax] is one reserved glyph,
  mapped to `⊥` in XMIR:

```
T > x
```

The terminator is a value with no data and no behaviour.
It may be named, stored, copied, and passed around like any other object;
  a copy of it is itself,
  and every attribute taken from it is itself again,
  so it flows silently through dispatch and application.
It detonates only when something *forces* it —
  tries to dataize it.
Then the program stops, for good:
  no object catches a forced bottom,
  because catching is not a thing anymore.

### An Epitaph, Not a Payload

A terminator may carry a *cause*:

```
T "this can't happen"
```

The first object applied to a bottom is remembered
  and printed when the bottom is finally forced.
That's all the cause does.
EO code can't read it back, match on it, or re-throw it —
  it is not an exception payload, it is an epitaph.

This is how the [`tuple`][tuple] object defines its `empty` companion:

```
tuple > empty
  T
    "Can't get tail from the empty tuple"
  T
    "Can't get head from the empty tuple"
  0
```

The head of the empty tuple *is* a terminator.
Building the tuple costs nothing and fails nothing;
  only the code that forces `empty.head`
  dies with the message above.

### Absent Means Bottom

Taking a void attribute that nobody filled
  [now yields][absent] a terminator instead of raising an error.
This one rule replaces the whole optional-argument machinery.
Consider [`i64.div`][div]:

```
[n x cant-divide] > div
  if. > @
    x.as-bytes.eq 00-00-00-00-00-00-00-00
    cant-divide
      "i64 cannot be divided by zero"
    ...
```

The third void, `cant-divide`, is a fallback the caller *may* supply.
If they don't, taking it yields `⊥`,
  the message lands in its cause slot,
  and division by zero terminates the program:

```
2.as-i64.div 0.as-i64
```

If they do, the same application invokes their handler instead:

```
2.as-i64.div
  0.as-i64
  "no luck" > [message]
```

This dataizes to `"no luck"`.
One mechanism, two behaviours, zero exceptions.

### One Escape Hatch

Sometimes a failure is expected and the show must go on.
For that single purpose there is the [`recovered`][recovered] atom:

```
recovered
  2.as-i64.div 0.as-i64
  7
```

It resolves the first attribute to its normal form;
  if that turns out to be a bottom, it behaves as the second,
  otherwise as the first.
This dataizes to `7`.
Note what `recovered` is not:
  it sees no cause, filters no "exception types," runs no `finally`.
When both attributes terminate, the result is still a bottom,
  which only an outer `recovered` may intercept —
  a bottom that nobody recovers ends the program.

### Tests Expect It Too

Since termination is now a first-class outcome,
  unit tests assert it with the throwing-test suffix `-->`,
  a sibling of the truthy `++>`:

```
2.as-i64.div 0.as-i64 --> stops-on-division-by-zero
```

The bottom was in 𝜑-calculus from the very first paper;
  `try` and `error` never were.
Now the language agrees with its own semantics,
  and the price of that agreement is one letter.

That's all for today.

[old]: https://news.eolang.org/2022-07-18-error-and-try-catch.html
[phi]: https://arxiv.org/abs/2111.13384
[issue]: https://github.com/objectionary/eo/issues/5261
[syntax]: https://github.com/objectionary/eo/issues/5264
[absent]: https://github.com/objectionary/eo/issues/5280
[div]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/i64/div.eo
[tuple]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/tuple.eo
[recovered]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/recovered.eo
