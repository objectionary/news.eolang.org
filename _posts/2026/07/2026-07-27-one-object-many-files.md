---
layout: post
date: 2026-07-27
title: "One Object, Many Files"
author: yegor256
---

Until recently, an object in EO was whatever a single `.eo` file said it was.
If you wanted `number` to know how to raise itself to a power,
  you either grew `number.eo` until nobody could read it,
  or you parked the operation in some other object with a made-up name.
We took a third road:
  an object and the package named after it are now the same thing,
  so `number` is defined by `number.eo` *and* by every file under `number/`.

<!--more-->

### The Problem

The EO runtime used to ship packages called `ms`, `tt` and `ss`.
They held mathematics, text and sequence helpers respectively,
  and their names were abbreviations that meant nothing to anyone
  who had not read the source.
Raising two to the fourth power looked like this:

```
ms.power 2 4
```

Trimming a string looked like this:

```
tt.trimmed "  hello  "
```

Every one of those names was an apology.
`ms` existed only because `power` had to live *somewhere*,
  and `number.eo` was already four hundred lines long.
The alternative was worse: fold `power`, `sqrt`, `sine`, `cosine`,
  `arc-sine`, `radians`, `degrees` and twenty more into `number` itself,
  and end up with an object whose definition nobody could hold in their head.

This is an old tension.
A small object is readable but poor.
A rich object is useful but unreadable.
Most languages resolve it by letting one type live in many files;
  EO had no such mechanism, because a file *was* an object.

### The Idea

Let the package do the work.

When the runtime is asked for an attribute that an object does not have,
  it looks for an object of that name
  in the package named after the object's type,
  and if it finds one, it binds the receiver as the first argument.
So this:

```
2.power 4
```

means exactly this:

```
number.power 2 4
```

Both forms compile, both run, and both give `16`.
The object `power` lives in the file `number/power.eo`,
  which declares `+package number` and defines a single object:

```
[num x] > power
  ...
```

Nothing was added to `number.eo`.
The `number` object is still a small thing that wraps some bytes.
But its surface now includes everything in the `number/` directory —
  twenty-five files at the time of writing,
  next to thirty-four for `string` and twenty-three for `tuple`.
The `ms`, `tt` and `ss` packages are gone.

The rule is uniform, so it works for operations of any arity.
`abs` takes one argument and reads as `-3.abs`:

```
[num] > abs
  if. > @
    value.gte 0
    value
    value.neg
  number num.as-bytes > value
```

While `contains` takes two and reads as `"hello".contains "h"`:

```
[text substring] > contains
  ...
```

The receiver is always the first argument.
There is no `self`, no `this`, and no special position —
  just a plain object whose first argument happens to be the thing
  you dispatched from.

### One File, One Object

The part we like most is what this does to the file tree.
EO keeps one object per file,
  and that rule did not bend to make this feature work.
Instead, the feature made the rule pay off:

```
number.eo
number/
  abs.eo
  cosine.eo
  power.eo
  sqrt.eo
  ...
```

Each extension is a file.
Each file has a name, a docblock, its own tests, and its own history in Git.
Adding an operation to `number` means adding a file, not editing one.
Two people extending `number` in the same week
  never touch the same lines,
  because they never touch the same file.

Nor is the directory reserved for us.
Any object gets this for free.
If you write `book.eo`, then `book/summary.eo` with `+package book`
  makes `(book "Dune").summary` work,
  and you never declare that fact anywhere —
  the directory name *is* the declaration.

### Under the Hood

Two things had to change.

First, attribute lookup.
`PhDefault` now tries the object's own package
  *before* it descends anywhere else.
Order matters here.
A `string` decorates `bytes`, and `bytes` has an `as-number`,
  so if the decoratee were consulted first,
  `"42.5".as-number` would parse those bytes as a raw number
  instead of reading the text.
The package goes first, then the λ of an atom, then the decoratee,
  and a name nobody knows terminates the computation.
When the package does answer, the receiver is bound into `α0` —
  the first positional attribute, not `ρ`.
That is what makes `2.power 4` and `number.power 2 4` the same expression:
  the implicit form fills `α0` for you,
  the explicit form leaves it for you to fill.

Second, the name `number` had to mean two things at once.
`PhNest` is the class that does it:
  a single shared instance that stands for both the package and the object.
Ask it for `power` and it hands out the extension.
Ask it for anything else, or copy it, and it collapses into the plain
  `number` object and delegates.
Because that instance is shared by every reference in the program,
  it refuses to be written to —
  `put` fails fast and tells you to `copy()` first.

This also broke a Java rule we had to route around.
A Java class and a Java package [cannot share a name][jls],
  and our generated code needed `Φ.number` and `Φ.number.power` side by side.
So generated classes keep the `EO` prefix
  while package segments now take `EO_`:
  `org.eolang.EO_number.EOpower`.
Ugly, invisible, and it works.

### Has Anyone Done This Before?

Parts of it, yes — and we would rather say so than pretend otherwise.

The rewrite itself is well-trodden.
[Uniform Function Call Syntax][ufcs] in [D][dlang-ufcs] and Nim
  turns `a.b(x)` into `b(a, x)` whenever `a` has no method `b`,
  which is precisely the transformation EO performs.
[Ada 2005's prefixed notation][ada-prefix] does the same with a tighter leash:
  `Y.Op(...)` stands for `P.Op(Y, ...)`,
  where `P` is the package in which the type of `Y` is declared.
[C# extension methods][csharp-ext] and [Kotlin extensions][kotlin-ext]
  reach the same syntax through static resolution and an import.

Spreading a type across files is equally old.
[Swift extensions][swift-ext] and [Objective-C categories][objc-cat]
  let a type's API grow in files far from its declaration.
Go simply defines methods anywhere in the package that owns the type.
Ruby leaves classes open.
[Smalltalk][pharo-ext] has had extension methods for decades —
  in Pharo you file a method under a protocol named after *your* package,
  and `String` grows a method that ships with your code, not with the kernel.
[C# partial classes][partial] merge several files into one class outright.
And Elixir has turned the naming into a convention:
  `String.trim/1`, `Enum.map/2` — a module named after the data it serves,
  subject always first.

What we have not found anywhere is the specific combination.
In every language above, the extension mechanism is announced:
  a `partial` keyword, an `extension` block, an `impl`, a `using` directive,
  a protocol whose name starts with an asterisk.
Something in the source says "this is an extension".
In EO nothing does.
The object and its namespace are not two entities that happen to be related —
  they are one name, resolved by one runtime lookup.
`number` is the object, `number` is the package,
  and the language never asks you which one you meant.
Add a file to the directory and the object grew a feature;
  delete it and the feature is gone.

We also could not find a language that pairs this
  with a one-object-per-file rule.
That pairing is what turns the feature from a convenience into a structure.
Elsewhere, "split a type across files" is a way to survive a big class.
Here, the class is never big:
  every operation is a file, always, with no threshold to cross
  and no decision to make about when to split.

So: novel as a whole, borrowed in every part.
We will take that.

### What It Cost

Honesty demands a note on the price.

Lookup is now a classpath probe,
  which is why the runtime caches the result of every probe it makes.
An attribute typo that used to fail immediately
  can now travel one hop further before it dies.
And a package that shares a name with an object means
  reading `number.power` requires knowing that `number` is both —
  a small tax on the reader, paid once.

We think a name that means one thing
  is worth more than a name like `ms` that means nothing.

That's all for today.
Next time an object of yours starts feeling crowded,
  make a directory instead of a longer file.

[jls]: https://docs.oracle.com/javase/specs/jls/se21/html/jls-6.html
[ufcs]: https://en.wikipedia.org/wiki/Uniform_function_call_syntax
[dlang-ufcs]: https://tour.dlang.org/tour/en/gems/uniform-function-call-syntax-ufcs
[ada-prefix]: https://www.adaic.org/resources/add_content/standards/05rat/html/Rat-2-3.html
[csharp-ext]: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods
[kotlin-ext]: https://kotlinlang.org/docs/extensions.html
[swift-ext]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/
[objc-cat]: https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html
[pharo-ext]: https://github.com/pharo-open-documentation/pharo-wiki/blob/master/General/Extensions.md
[partial]: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods
