---
layout: post
date: 2026-07-17
title: "Encapsulation by Obfuscation"
author: yegor256
---

A while ago we [explained][auto-names]
  what the `>>` syntax stands for:
  it auto-names an abstract object when a meaningful name is unnecessary,
  and the compiler quietly generates a random unique one under the hood.
Recently we let this same `>>` carry a handle of your choosing,
  and this small change gave EO a new way to hide things.
We call it "encapsulation by obfuscation".

<!--more-->

### The Problem

Most languages guard a name with a keyword.
Java has [`private`][java-access] and Swift has [`fileprivate`][swift-access];
  Python merely [asks you nicely][pep8] with a leading underscore.
In each case the attribute keeps its original name,
  and enforcement ranges from a compiler error to a matter of etiquette.
The name is still there,
  still spelled the way the author spelled it,
  and a sufficiently motivated caller (reflection, a mangling trick, a cast)
  can almost always reach it.

EO has no visibility keywords, and we did not want to add any.
Yet a `.eo` file often needs a helper object
  that makes sense only inside that file
  and should never be addressed from anywhere else.
How do you make something private in a language that has no word for "private"?

### The Idea

You cannot refer to a name you do not know.
That is the whole trick.

Consider an object that clamps a number to at most ten,
  leaning on a small helper to take the absolute value first:

```
[n] > out
  if. > @
    (abs n).gt 10
    10
    abs n
  [x] > abs
    if. > @
      x.gt 0
      x
      x.neg
```

The `abs` object is a true helper.
It exists only to serve `out`, and it means nothing on its own.
Yet, declared with a plain `> abs`,
  it becomes a public attribute of `out`,
  and any other file can reach it:

```
(out 42).abs
```

That call is nonsense, but the compiler happily allows it,
  because `abs` is a visible name.
We would rather it were impossible.
So we change one character:

```
[x] >> abs
```

Now `abs` is a file-local handle.
Inside the file, `out` still calls it exactly as before.
Outside the file, `abs` simply does not exist,
  and `(out 42).abs` no longer compiles.

### Under the Hood

The compiler treats `abs` as nothing more than a temporary label.
During parsing it assigns the object a deterministic "cactus" name
  derived from the line and column where it was declared,
  collects a per-file table of handles,
  and rewrites every mention of `abs` to that cactus name.
The name it picks is `a🌵6-3` —
  an `a`, the cactus emoji, and the coordinates of the declaration.
What comes out the other end looks roughly like this:

```
[n] > out
  if. > @
    (a🌵6-3 n).gt 10
    10
    a🌵6-3 n
  [x] > a🌵6-3
    if. > @
      x.gt 0
      x
      x.neg
```

The helper is still there, still perfectly usable —
  `out` clamps its argument exactly as before,
  because both call sites were rewritten to the same cactus name.
But its name now contains 🌵,
  and the cactus emoji is deliberately excluded
  from the grammar of a valid identifier.
No programmer can type it, in this file or any other.
Another file that tries `(out 42).abs` finds no `abs`;
  and it cannot ask for `a🌵6-3` either,
  because it neither knows the coordinates nor is allowed to spell the cactus.
The attribute is not hidden behind a rule that says "do not look".
It is hidden because there is nothing left to look for.

### Why "Obfuscation"

We are not pretending this idea is new;
  it is inherited from decades of language design.
Lisp's [`gensym`][gensym] has minted unwriteable symbols since the 1970s,
  [hygienic macros][hygienic] rename identifiers precisely
  so surrounding code cannot capture them,
  [Python's double underscore][py-mangling] mangles a name into a different one,
  and every JavaScript [minifier][terser] renames locals
  so the outside world cannot reach them.
The [object-capability model][ocap] makes the same wager at runtime:
  authority you were never handed is authority you cannot name.
What EO does is take that old idea — privacy through an unspeakable name —
  and make it the language's only encapsulation mechanism,
  operating at file scope, with no keyword and no annotation.
The compiler obfuscates the name; the obfuscation *is* the encapsulation.
Hence the honest label.

The pleasant part is that it costs the programmer nothing.
You write a friendly `abs` and read a friendly `abs` everywhere in your file.
The obfuscation happens once, at compile time,
  and only the compiler ever sees the ugly name.
You get the readability of a normal identifier and the privacy of a secret one,
  without a single access modifier.

That's all for today.
Give `>> name` a try the next time a helper object should stay at home.

[auto-names]: https://news.eolang.org/2025-02-21-auto-named-abstract-objects.html
[java-access]: https://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html
[swift-access]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/
[pep8]: https://peps.python.org/pep-0008/
[gensym]: http://www.lispworks.com/documentation/HyperSpec/Body/f_gensym.htm
[hygienic]: https://en.wikipedia.org/wiki/Hygienic_macro
[py-mangling]: https://docs.python.org/3/tutorial/classes.html#private-variables
[terser]: https://terser.org
[ocap]: https://en.wikipedia.org/wiki/Object-capability_model
