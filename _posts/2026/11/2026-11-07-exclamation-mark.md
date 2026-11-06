---
layout: post
date: 2026-11-07
title: "What Does the Exclamation Mark Mean?"
author: yegor256
---

In EO, an exclamation mark after a binding asks for a dataized result:

```eo
5 > five!
```

The name is `five`; `!` is syntax attached to the binding, not part of its name.
The important consequence is that `five` behaves as **bytes**, rather than
preserving the interface of the original number. The spelling is small,
but its history makes it easy to read too much into it.

<!--more-->

### Read the expansion first

The [parser transformation][current-parser] expands the binding above into:

```eo
(dataized 5).as-bytes > five
```

Dataization retrieves an object's data. The `dataized` object turns that data
into a bytes object, and `.as-bytes` selects the bytes result. This is not
Boolean negation or a command to execute a statement immediately.

The [parser fixture][parser-test] includes `5 > num!` and checks that the
result contains a `dataized` application under `.as-bytes`, with the binding
name retained and the internal `const` marker removed. Thus, `const` in the
parser's intermediate representation does not mean that EO acquired a
separate kind of mutable variable or a new runtime primitive.

For a concrete result, the [runtime examples][runtime-examples] check that
dataizing `5` produces `40-14-00-00-00-00-00-00`, the bytes of its double
representation. To use those bytes as a number again, select `.as-number`:

```eo
5 > five!
five.as-number > number
```

This example combines the binding from the parser fixture with the
`as-number` conversion demonstrated in the runtime's `dataized` documentation.
It illustrates the result's interface: bytes are not automatically a number
with all the original number's attributes. The same runtime examples compare
dataizing `42` with `42.as-bytes`, and dataizing `"hello"` with
`"hello".as-bytes`.

### What changed in 0.36.0?

The [0.36.0 release post]({{ "/2024-04-16-release-0-36-0.html" | relative_url }}#caching)
describes two historical meanings. Before that release, `!` was intended to
cache attribute calculations. In 0.36.0, the description changed to dataizing
the object on its first touch and forwarding subsequent operations to the
resulting bytes. The release also made `!` syntax sugar for `dataized`.
Both the [tagged parser transformation][old-parser] and the
[tagged runtime declaration][old-runtime] confirm the bytes expansion in
0.36.0:

```eo
some-object > cached!
# The expansion in 0.36.0:
(dataized some-object).as-bytes > cached
```

Here `some-object` is a placeholder for an object that can be dataized, not
a standard library object to copy into a program. The name `cached` comes
from the historical explanation; it is not a special identifier.

### How should we read it today?

For the current-source comparison, this article uses EO master commit
[`94d598975a`][snapshot], dated November 6, 2026, whose project version is
`1.0-SNAPSHOT`. This is a development snapshot, not a claim about every
published EO version. The parser still performs the same bytes expansion,
even though implementation details and syntax elsewhere in EO have evolved.

The old promise about first-touch caching needs separate care. In this
snapshot, the JVM atom [`EOdataized.lambda()`][current-atom] constructs a
bytes value by calling `new Dataized(...).take()` each time that method is
invoked. [`PhDefault.take()`][current-dispatch] dispatches an atom's lambda,
and [`AtComposite.get()`][current-attribute] evaluates its expression on each
access. These methods do not themselves retain a first result. Consequently,
the historical release text should not be used as a blanket guarantee that
repeated accesses to any `!` binding evaluate a side effect exactly once in
current EO. The expansion establishes dataization and a bytes interface;
evaluation count also depends on the compiler, runtime, and how the result
is retained and accessed.

When reading old EO code, check its EO version before relying on the word
"cached." When writing code for the inspected snapshot, start with the
explicit `dataized` expansion and choose the conversion needed by the next
operation. That explains what `!` does without silently carrying an old
caching guarantee into a newer runtime.

[current-parser]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-parser/src/main/resources/org/eolang/parser/parse/const-to-dataized.xsl
[parser-test]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-parser/src/test/resources/org/eolang/parser/eo-packs/parse/const-to-dataized.yaml
[runtime-examples]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/eo/dataized.eo
[old-parser]: https://github.com/objectionary/eo/blob/0.36.0/eo-parser/src/main/resources/org/eolang/parser/const-to-dataized.xsl
[old-runtime]: https://github.com/objectionary/eo/blob/0.36.0/eo-runtime/src/main/eo/org/eolang/dataized.eo
[snapshot]: https://github.com/objectionary/eo/tree/94d598975a45bb3089a85c4d6bc19c5a0b0343b1
[current-atom]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/java/org/eolang/EOdataized.java
[current-dispatch]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/java/org/eolang/PhDefault.java
[current-attribute]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/java/org/eolang/AtComposite.java
