---
layout: post
date: 2026-08-13
title: "Can and Cannot"
author: yegor256
---

A unit test in EO is [an attribute][attrs] of the object it tests,
  and it comes in exactly two kinds:
  the object can do something, or it cannot.
Two arrows spell the difference, `++>` and `-->`,
  and since the end of July a lint refuses to let the name of a test
  disagree with the arrow that declares it.
It reads like bureaucracy.
It is the cheapest design review we have.

<!--more-->

### The Negative Half

Most testing frameworks treat failure as an afterthought.
In JUnit, "this works" is an expression,
  while "this dies" is an expression wrapped in a lambda
  and handed to `assertThrows`.
In Python, the first is a line, the second is a `with` block.
The asymmetry is small, but it is charged on every negative test,
  and the tests nobody writes are always the ones that cost extra.

In EO both cost one arrow.
Here is the whole test that `i64` refuses to divide by zero,
  from [`i64/div.eo`][div]:

```
2.as-i64.div 0.as-i64 --> stops-on-division-by-zero
```

The `-->` suffix declares a *throwing test attribute* ([spec][spec], §6.3):
  sugar for `[] -> name`, exactly as `++>` is sugar for `[] +>`.
Whatever stands to the left becomes the `@` of a parameterless formation,
  and the test passes when dataizing that formation
  [terminates][terminator].
The transpiler reads one character —
  the `-` in front of the attribute name —
  and emits `assertThrows` where it would otherwise emit `assertTrue`.

Nothing changes when the test needs more room.
The arrow can head its own line, with the body indented below,
  as in [`malloc.eo`][malloc]:

```
malloc.for --> stops-on-overflowing-a-string-block
  "Hello"
  m.put > [m]
    "Much longer string!"
```

There is no `expected`, no `catch`, no exception class to name.
There is nothing to name, because
  [a forced bottom carries no payload][terminator]
  that a test could match on.
The assertion is the arrow.

### The Name Must Agree

Since [29 July][lint] the `bad-test-name` lint checks
  the name of every test against the arrow that declares it:

- a positive `+>` test must be named `can-…` or `accepts-…`;
- a negative `->` test must be named `cannot-…`, `rejects-…` or `stops-on-…`.

So this is a defect, and the lint says so:

```
front --> can-take-a-fractional-index
```

Note what is being checked.
Not that the name matches *some* approved prefix —
  that the prefix matches *this* arrow.
A negative test wearing a positive name is a contradiction
  between two halves of the same line,
  and one of the halves is wrong.
Usually it is the arrow, pasted from the test above it.

### Why a Prefix Deserves a Lint

The obvious reason is uniformity.
`eo-runtime` holds 1479 positive tests and 147 negative ones;
  all but one of the positives begin with `can-`,
  and 141 of the negatives begin with `stops-on-`,
  because termination is what the runtime does when asked for nonsense.
A reader scrolling through [`recovered.eo`][recovered]
  meets a column of sentences that all open the same way,
  and a `grep` for `stops-on` returns every negative test in the language.

The better reason is discipline.
A prefix is a grammatical obligation:
  it makes the object the subject of a sentence
  that the author now has to finish.
"`malloc` stops on overflowing a string block."
"`recovered` can keep the value when not terminated."
Names like `test1`, `works`, `bug-4212` and `it-works`
  cannot survive that requirement,
  because none of them is the tail of a sentence about anything.

The obligation pushes back on the test, too.
If the sentence needs an "and" to finish,
  the test is checking two behaviours and should be two tests.
If no subject fits in front of it,
  the test is not about the object it lives in.
If the sentence can only be finished by restating the code —
  "can call `put` with a long string" —
  then the author knows what the test does
  but not what it proves.
Naming is where a bad test declares itself first,
  earlier than coverage reports and much earlier than a failure,
  and a prefix is what makes the declaration audible.

None of this is new advice; every team has a naming convention.
The difference is where it lives.
A convention in a style guide is enforced by whoever reviews the pull request,
  on the days they have the patience.
This one is enforced by the build, in every project that compiles EO,
  which is why the prefix list is short and closed:
  five words, no per-project dialects.

### The Rest of the Rules

The naming lint has siblings.
`unit-test-missing` complains when an object has no tests at all,
  `unit-test-without-phi` when a test binds attributes but never `@`,
  and `wrong-test-order` when tests sit above the code they verify
  instead of below it.
The parser adds its own: a test attribute is legal only as a direct child
  of the top-level object, exactly one blank line must precede it,
  and an atom's body may contain nothing but its voids and its tests
  ([spec][spec], R-6.3.1, R-6.3.3, R-6.5.3).

Together they leave one shape for a unit test,
  which is the point.
A test name is the only documentation guaranteed to be read
  at the exact moment it matters — when it fails —
  and rigidity is cheap in that spot.

That's all for today.

[attrs]: https://news.eolang.org/2026-07-31-tests-are-attributes.html
[terminator]: https://news.eolang.org/2026-07-31-the-terminator.html
[spec]: https://github.com/objectionary/eo/blob/master/eo-parser/PARSER_SPEC.md
[lint]: https://github.com/objectionary/lints/blob/master/src/main/resources/org/eolang/motives/tests/bad-test-name.md
[div]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/i64/div.eo
[malloc]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/malloc.eo
[recovered]: https://github.com/objectionary/eo/blob/master/eo-runtime/src/main/eo/recovered.eo
