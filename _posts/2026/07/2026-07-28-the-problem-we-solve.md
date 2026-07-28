---
layout: post
date: 2026-07-28
title: "The Problem We Solve"
author: yegor256
---

People keep asking us why EO exists,
  since the world hardly needs yet another programming language.
It is a fair question, and it deserves a straight answer.
EO is not trying to out-comfort Java or out-hype Rust.
It is a research instrument pointed at one old, specific,
  and still unsolved problem:
  object-oriented code is pleasant for programmers to write
  and expensive for computers to run.

<!--more-->

### The Problem

Objects earn their keep at design time.
Abstraction, encapsulation, and polymorphism
  let us hold a large system in a small head.
The bill arrives at run time.
[Craig Chambers and David Ungar][chambers1991] put it plainly back in 1991:
  object-oriented languages "contain a number of features
  that make programs easier to write but slower to run."
[Urs Hölzle][holzle1991] explained why compilers struggle to help:
  procedures are smaller and calls more frequent than in C or Fortran,
  and dynamic dispatch hides the target of every call,
  so inlining and interprocedural analysis simply do not apply.
[Karen Driesen and Urs Hölzle][driesen1996] then measured the damage:
  C++ programs spend a median of 5.2% and up to 29% of their time
  executing dispatch code,
  and when every function is virtual,
  the median rises to 13.7% and the maximum to 47%.
In dynamic languages such as Python the same story is told louder,
  since there an attribute access is a dictionary lookup
  and a method call is a small ceremony.

We once [measured this ourselves][fibonacci],
  implementing the same Fibonacci algorithm twice in each language:
  once with functions, once with objects.
The table shows millions of CPU instructions
  spent computing the 32nd Fibonacci number:

| Language | w/functions | w/objects | Ratio |
|----------|------------:|----------:|------:|
| Java     | 41          | 4,589     | 109x  |
| C#       | 36          | 5,785     | 157x  |
| C++      | 93          | 7,203     | 76x   |
| Go       | 39          | 15,907    | 403x  |

Same algorithm, same machine, same compiler —
  the only difference is objects instead of functions.
This is the tax we pay for abstraction,
  and we pay it on every core, in every data center, every day.

### Forty Years of Workarounds

The problem is not new, and the people who attacked it were not weak.

[Peter Deutsch and Allan Schiffman][deutsch1984] opened the campaign in 1984
  with inline caching in Smalltalk-80:
  remember where the last lookup landed,
  and in nine cases out of ten the next one lands there too.
[Chambers and Ungar][chambers1989] answered in 1989 with customization in SELF:
  compile a separate copy of each method for each receiver type,
  so that within a copy the type is a compile-time constant.
[Hölzle][holzle1991] generalized the cache in 1991
  into polymorphic inline caches that hold several targets per call site,
  then in 1994 added [type feedback][holzle1994]:
  let the runtime observe actual receiver types
  and feed them back to the compiler,
  which inlines the hot target behind a guard.
[Jeff Dean, David Grove, and Craig Chambers][dean1995]
  gave compilers class hierarchy analysis in 1995;
  [David Bacon and Peter Sweeney][bacon1996] sharpened it in 1996
  into rapid type analysis,
  which prunes the call graph down to classes actually instantiated.
[Julian Dolby][dolby1997] inlined whole objects into their containers in 1997;
  [Jong-Deok Choi and colleagues][choi1999] brought escape analysis to Java
  in 1999,
  so that objects provably local to a method could live on the stack;
  [Kazuaki Ishizaki and colleagues][ishizaki2000] devirtualized calls
  with code patching in 2000;
  [Christian Wimmer and Hanspeter Mössenböck][wimmer2010]
  fused parent and child objects at run time in 2010;
  [Anders Møller and Oskar Veileborg][moller2020] eliminated
  the abstraction overhead of Java stream pipelines in 2020.

This list is a fraction of the literature —
  a fuller chronology is what our [papers page][papers] and lectures
  keep growing.
Yet, to the best of our knowledge,
  only a few of these techniques run inside Clang and OpenJDK today.

### Why So Little Sticks

Because in Java, C++, and Python, almost every one of those optimizations
  is a bet the language allows to lose.

A class loaded at run time can extend the hierarchy
  after class hierarchy analysis has finished,
  so a devirtualized call needs a guard and machinery to undo itself.
Reflection can reach a method that no static analysis ever saw,
  so dead code is never provably dead.
Mutation means a fact proven about an object at one program point
  has expired by the next.
Even an innocent annotation can break an optimizer:
  inline a method into its caller,
  and whatever semantics a framework attached to `@Transactional`
  silently disappears.

Modern JITs cope by speculating:
  optimize as if the world were simple,
  check at run time that it still is,
  and deoptimize when it is not.
HotSpot is a genuine marvel of this kind —
  but it treats symptoms.
The language keeps making promises to the programmer
  that the compiler cannot verify,
  so the compiler hedges,
  and every hedge is a check, a guard, a table, a pause.

### The Workaround

Programmers did not wait for the papers to be written.
Decades of profiling taught the OO community one practical lesson —
  objects are slow —
  and the community answered with a silent agreement:
  avoid them.

Open any performance-minded Java codebase
  and you will find ALGOL wearing Java syntax:
  [static methods][statics] instead of objects,
  [NULL][null] instead of an object that says "not found",
  [global state][globals] instead of composition,
  primitives and arrays instead of types,
  singletons, utility classes, empty constructors,
  and allocation treated as a sin.
None of this is ignorance.
It is self-defense:
  the compiler cannot make objects cheap,
  so programmers make them rare.

Every such trick buys speed and pays with design.
[David West][west] called the result
  "a pale shadow of the original idea" of objects,
  and the shadow is what we now maintain:
  code that reads poorly, changes reluctantly,
  and greets every refactoring with a production bug.
The performance problem of OOP has quietly become
  a quality problem of software.

### Our Bet

EO takes the other road: change the language, not the compiler.

EO is a small object-oriented language
  whose semantics is [𝜑-calculus][phi],
  a formalism where a program is nothing but objects:
  formation, application, dispatch, and data.
There are no classes, no inheritance, no statics, no null,
  no mutation, no reflection, no annotations,
  and no flow-control statements —
  `if`, `while`, and `seq` are objects too.

We expect any object-oriented language to be reducible to EO.
This is not a hypothesis we sit on:
  [jeo][jeo] already disassembles Java bytecode into EO,
  and [hone][hone] runs the full round trip —
  Java to bytecode to EO to 𝜑-expressions,
  then normalization by [formal rewriting rules][normalizer]
  whose confluence is [proved in Lean][proof],
  then back to bytecode.
The [preliminary results][benchmark] are promising:
  we reproduced the stream-pipeline fusion of Møller and Veileborg,
  but as rewriting rules over 𝜑-expressions
  rather than a special-purpose bytecode analysis.

Since EO is simple, the reduction target is simple,
  and this is where the strategy points:
  build a compiler — perhaps with a virtual machine behind it,
  perhaps one day with a dataflow processor under it —
  that turns EO into binaries
  running faster than what javac, gcc, or CPython
  produce from the original source.

### Three Reasons to Believe

Our belief stands on facts about EO
  that Java, C++, and Python cannot offer.

First, all objects are immutable.
An attribute is bound once and never reassigned,
  so any expression may be computed once and its result shared,
  caching is always sound,
  and aliasing analysis is unnecessary
  because there are no writes to alias.
What escape analysis spends its budget proving about a Java object,
  the EO grammar simply forbids to be otherwise.

Second, type inference is total.
There is no reflection, no type casting, and no run-time class loading,
  so the object graph is closed at compile time
  and every dispatch resolves statically to one target.
No virtual tables, no guarded devirtualization —
  a call compiles to a jump,
  and nothing in the language can invalidate it later.
Class hierarchy analysis and rapid type analysis speculate about this world;
  EO is this world by construction.

Third, there is no global state.
Every block of memory in EO is strictly scoped:

```
malloc.of
  8
  [m]
    m.put 42 > @
```

The eight bytes exist only inside the scope object,
  and when its dataization ends, the block is reclaimed.
An object's lifetime is written in the source,
  escape analysis reads directly off the program text,
  and leakage is impossible
  because there is no global to park a reference in.

While building [the compiler][eo] we keep finding smaller gifts
  of the same nature.
Control flow is objects,
  so a program is one expression graph
  that a dataflow machine can evaluate without reconstructing
  a control-flow graph.
There is exactly one data primitive — `bytes` —
  and `number`, `string`, and `bool` merely decorate it,
  so rewriting rules stay uniform.
Decoration replaces inheritance,
  so "super" is just another attribute, statically known.
And since every object lives in its own file
  and a package is itself an object,
  whole-program analysis actually sees the whole program.

### What We Cannot Claim Yet

We have no proof — only the bet, the pipeline, and early numbers.
The language is still moving:
  its grammar, its semantics, and the calculus itself
  are being revised as we learn.
Normalization may inflate code the way aggressive inlining
  [famously did][holzle1994];
  reduced Java that leans on reflection may arrive in EO
  stripped of the very guarantees we brag about;
  the dataflow hardware may remain a patent portfolio.
These are the risks of a research direction, and we accept them.

For forty years, compilers have been taught to guess
  what object-oriented programs will do
  and to apologize at run time when they guess wrong.
Our wager is that the guessing was never necessary —
  it was the languages that made it so.
EO is our attempt to build the language that doesn't.

That's all for today.

[chambers1991]: https://dl.acm.org/doi/10.1145/117954.117955
[holzle1991]: https://link.springer.com/chapter/10.1007/BFb0057013
[driesen1996]: https://dl.acm.org/doi/10.1145/236337.236369
[deutsch1984]: https://dl.acm.org/doi/10.1145/800017.800542
[chambers1989]: https://dl.acm.org/doi/10.1145/73141.74831
[holzle1994]: https://dl.acm.org/doi/10.1145/178243.178478
[dean1995]: https://link.springer.com/chapter/10.1007/3-540-49538-X_5
[bacon1996]: https://dl.acm.org/doi/10.1145/236337.236371
[dolby1997]: https://dl.acm.org/doi/10.1145/258915.258861
[choi1999]: https://dl.acm.org/doi/10.1145/320384.320386
[ishizaki2000]: https://dl.acm.org/doi/10.1145/353171.353191
[wimmer2010]: https://dl.acm.org/doi/10.1145/1668243.1668245
[moller2020]: https://dl.acm.org/doi/10.1145/3428236
[fibonacci]: https://github.com/yegor256/fibonacci
[statics]: https://www.yegor256.com/2014/05/05/oop-alternative-to-utility-classes.html
[null]: https://www.yegor256.com/2014/05/13/why-null-is-bad.html
[globals]: https://www.yegor256.com/2018/07/03/global-variables.html
[west]: https://www.oreilly.com/library/view/object-thinking/0735619654/
[papers]: https://news.eolang.org/papers.html
[phi]: https://arxiv.org/abs/2111.13384
[jeo]: https://github.com/objectionary/jeo-maven-plugin
[hone]: https://github.com/objectionary/hone-maven-plugin
[normalizer]: https://github.com/objectionary/eo-phi-normalizer
[proof]: https://github.com/objectionary/proof
[benchmark]: https://github.com/objectionary/benchmark
[eo]: https://github.com/objectionary/eo
