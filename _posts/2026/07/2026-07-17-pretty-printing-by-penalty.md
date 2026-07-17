---
layout: post
date: 2026-07-17
title: "Pretty-Printing by Penalty"
author: yegor256
---

Ask two EO programmers to lay out the same object
  and you may well get two different files.
One stacks everything vertically, another folds it onto a single line,
  and both are correct.
We would rather the compiler decided,
  and always arrived at the same, prettiest answer.
So we taught it to score every possible layout
  and pick the one that hurts the least.

<!--more-->

### The Problem

Formatting is a small tax that every project pays forever.
Curly-brace languages hand the bill to tools like
  [Prettier][prettier], [gofmt], and [clang-format],
  each carrying hundreds of options and special cases,
  because the layout space of C or JavaScript is enormous
  and full of exceptions.
EO has a much stricter grammar,
  yet the same ambiguity remains at the small scale:
  an object can be spelled tall or wide,
  and nothing in the language says which is nicer.
We wanted one canonical form, chosen automatically,
  with no table of options to argue about.

### The Idea

The trick is to stop asking "which layout is correct?"
  and start asking "which layout is cheapest?"

Give every symbol in the output a penalty.
As a [first][#5430] cut:

* an indent costs 3 points,
* an open bracket costs 7 points,
* every character past the 80th column costs 1 point,
* everything else is free.

Then render the object in all of its valid layouts,
  add up the penalty for each,
  and keep the one with the smallest total.

Here is an object spelled vertically.
It has five indents, so its penalty is 15:

```
[] > foo
  gt. > @
    42
    bar.hello 88
```

The same object, folded once, drops to two indents,
  for a penalty of 6:

```
gt. > [] > foo
  42
  bar.hello 88
```

And squeezed onto one line, it pays for a single bracket instead,
  a penalty of 7:

```
42.gt (bar.hello 88) > [] > foo
```

Six beats seven beats fifteen, so the middle layout wins.
Change the weights and you change EO's taste in code,
  but the machinery never changes:
  enumerate, score, choose the minimum.

### Why EO Is a Perfect Fit

This approach is unusually comfortable in EO,
  for reasons that would frustrate it in most languages.

*The meaning lives in the tree, not in the whitespace.*
An EO program is a 𝜑-calculus expression,
  and its layout is pure presentation.
There are no semicolons to place, no braces to align,
  and comments live only at the top of the file,
  never wedged between two lines the printer might want to merge.
The formatter may re-lay-out freely,
  certain that no arrangement can change what the program means.

*The grammar is small and regular.*
Almost everything in EO is the same shape —
  an abstraction with attributes, or an application of one object to others.
A single penalty function therefore covers the whole language,
  with none of the per-construct exceptions
  that bloat a `clang-format` configuration.

*The layout space is finite and shallow.*
Each object is essentially either vertical or horizontal,
  so the number of candidate layouts for a subtree stays modest,
  and a recursive search can weigh them all
  without the combinatorial blow-up
  a curly-brace language would suffer.

*There is a genuine appetite for one true form.*
The EO community already prefers a single canonical style.
A penalty function turns that preference into arithmetic,
  and arithmetic does not hold opinions.

### Prior Art

We are not claiming to have invented this.
Treating layout as a cost to be minimized is an old and honorable idea.

The seminal work is Knuth and Plass's
  [line-breaking algorithm][knuth-plass] from 1981,
  which assigns "badness" to each way of breaking a paragraph
  and uses dynamic programming to find the globally optimal set of breaks;
  it is why TeX sets such beautiful paragraphs.
The classic pretty-printers by [Oppen][oppen] and later [Wadler][wadler]
  chose layouts greedily instead,
  trading optimality for speed.

The closest relative to what we are doing is
  Phillip Yelland's [*A New Approach to Optimal Code Formatting*][yelland] (Google, 2016),
  whose `rfmt` formatter selects the layout
  that minimizes an explicit, tunable cost function —
  penalizing characters past the margin and rewarding compactness —
  exactly the shape of our penalty.
Jean-Philippe Bernardy's [*A Pretty But Not Greedy Printer*][bernardy] (ICFP 2017)
  makes the "render everything, keep the shortest that fits" idea rigorous,
  and Porncharoenwase and colleagues'
  [*A Pretty Expressive Printer*][expressive] (OOPSLA 2023)
  generalizes the cost into a pluggable "cost factory,"
  of which our indent/bracket/overflow weights are simply one instance.

Our contribution is not the algorithm
  but the marriage:
  a strict-grammar language whose finite, semantics-free layout space
  lets a penalty-minimizing printer do its best work
  with almost no special cases.
The idea is borrowed; the fit is ours.

That's all for today.
Next time your EO looks a little off,
  remember that somewhere the compiler is quietly adding up the damage,
  and choosing the layout that costs you the least.

[prettier]: https://prettier.io
[gofmt]: https://pkg.go.dev/cmd/gofmt
[clang-format]: https://clang.llvm.org/docs/ClangFormat.html
[knuth-plass]: https://onlinelibrary.wiley.com/doi/10.1002/spe.4380111102
[oppen]: https://dl.acm.org/doi/10.1145/357114.357115
[wadler]: https://homepages.inf.ed.ac.uk/wadler/papers/prettier/prettier.pdf
[yelland]: https://research.google/pubs/pub44667/
[bernardy]: https://dl.acm.org/doi/10.1145/3110250
[expressive]: https://dl.acm.org/doi/10.1145/3622837
[#5430]: https://github.com/objectionary/eo/issues/5430
