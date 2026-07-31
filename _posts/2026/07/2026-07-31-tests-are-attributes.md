---
layout: post
date: 2026-07-31
title: "Tests Are Attributes"
author: yegor256
---

Most languages keep unit tests at a distance from the code they verify:
  a parallel source tree, a mirror class with a `Test` suffix,
  a framework that finds tests by annotation or naming convention.
The distance is the problem:
  mirrors go stale, tests drift,
  and "where are the tests for this?" becomes a question worth asking.
In EO we reduced the distance to zero.
A test is an attribute of the object it tests,
  declared inside the object, in the same file, in the same scope.

<!--more-->

### One File, No Mirror

Here is an object with two tests:

```
[x] > square
  x.times x > @

  eq. ++> can-square-a-negative
    square -3
    9

  eq. ++> can-square-zero
    square 0
    0
```

The `++>` suffix declares a *test attribute* ([spec][spec], §6.3):
  the expression to its left is the whole body of the test,
  and the test passes when that expression dataizes to `TRUE`.
There is a second kind, `-->`, for when termination is the expected outcome —
  we met it [earlier today][terminator]:

```
2.as-i64.div 0.as-i64 --> stops-on-division-by-zero
```

That's the entire arsenal:
  a test either evaluates to truth or is expected to die.
No runner in sight, no annotations, no imports;
  even the assertion "library" is just `eq.`, `lt.`, and friends —
  ordinary objects from the runtime.

### An Attribute Like Any Other

The suffix `++> name` is sugar for `[] +> name`:
  a parameterless formation, bound to the enclosing object
  under a name marked as a test.
In XMIR the test is a binding like any other,
  distinguished only by a `+` (or `-`) in front of its name.
The compiled object carries its tests
  the way it carries every other attribute,
  and running one means taking that attribute and dataizing it —
  which is precisely what the generated JUnit method does,
  one `assertTrue` (or `assertThrows`) per marked binding.

Being an attribute designs the test for you.
A test is a formation whose `@` is a single expression,
  so its verdict is one boolean, not a report of several.
It has no parameters and no shared fixtures;
  whatever it needs, it names inside itself,
  as [`i64.div`][div] does when checking division by one:

```
++> can-divide-by-one
  eq. > @
    dividend.div 1.as-i64
    dividend
  -235.as-i64 > dividend
```

And since tests are siblings of the object's internal attributes,
  they may reach helpers directly —
  the "should I test private methods?" debate
  has nothing left to attach to.

### Atoms Sign a Contract

Some objects are *atoms*: their bodies live in Java or JavaScript, not in EO.
An atom's EO body may contain nothing
  but its void attributes and its tests ([spec][spec], R-6.3.1).
Here is [`recovered`][recovered], abridged:

```
[] > recovered /A
  ? > value /A?
  ? > alternative /A

  eq. ++> can-fall-back-to-the-alternative-when-terminated
    recovered
      2.as-i64.div 0.as-i64
      7
    7

  eq. ++> can-keep-the-value-when-not-terminated
    recovered
      42
      7
    42
```

The implementation is elsewhere;
  what the EO file states is the signature and the behaviour —
  an executable contract, sitting exactly where a reader looks first.

### Who Did It First?

Putting tests *near* code is not new.
[D][d] compiles `unittest` blocks written inside classes and structs;
  [Pyret][pyret] attaches a `where:` clause of examples to a function;
  Python runs examples found in docstrings with [doctest][doctest],
  and Rust does the same with [documentation tests][rust];
  Clojure's [`with-test`][clojure] hangs a test function on a var's metadata.
Yet in every one of these, the test is a guest:
  a block, a clause, a comment, a metadata entry —
  something the toolchain knows how to run
  but the language does not know how to hold.
In EO the test is an object,
  and an attribute of the object it tests:
  present in the object model, in the XMIR, in the compiled artifact,
  reachable by `take` like everything else.
As far as we know, no other language does this.
If we are wrong, tell us in the comments —
  we would genuinely like to meet the precedent.

That's all for today.

[spec]: https://github.com/objectionary/eo/blob/master/eo-parser/PARSER_SPEC.md
[terminator]: https://news.eolang.org/2026-07-31-the-terminator.html
[div]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/i64/div.eo
[recovered]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/recovered.eo
[d]: https://dlang.org/spec/unittest.html
[pyret]: https://pyret.org/docs/latest/testing.html
[doctest]: https://docs.python.org/3/library/doctest.html
[rust]: https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html
[clojure]: https://clojuredocs.org/clojure.test/with-test
