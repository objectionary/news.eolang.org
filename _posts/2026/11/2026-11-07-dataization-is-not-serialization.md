---
layout: post
date: 2026-11-07
title: "Dataization Is Not Serialization"
author: yegor256
---

If an EO object has three attributes, does dataization turn all three into bytes?
[This question][question] exposes a common misunderstanding.
Dataization obtains the data that an object represents.
Serialization encodes a chosen representation of an object's state,
with rules for such things as field names, ordering, and nested objects.
The `dataized` object does not supply such an encoding.

<!--more-->

The question was asked about EO 0.36.0, when [`dataized`][dataized] was introduced.
The examples below use that version's syntax and runtime behavior;
later releases have changed syntax and failure handling.
The distinction between obtaining data and encoding an object's structure remains.

### Follow the Data

An object may have a `Δ` (Delta) asset containing bytes.
The [runtime's dataization procedure][delta] first looks for attached data.
Otherwise, for an atom, it evaluates the atom and dataizes the resulting object.
Otherwise, if the object has `@` (the decoration attribute `φ`),
it dataizes the decorated object.
If none of these routes is available, dataization fails.
The [post about special attributes and assets][assets] explains these terms.

There is no step that walks through every named attribute and concatenates its data.
Consider an explicit byte sequence, so that numeric encoding is out of the picture:

```eo
[] > five
  00-00-00-00-00-00-00-05 > @
  [] > ignored-attribute

(dataized five).as-bytes > result
```

When `result` is dataized, its bytes are `00-00-00-00-00-00-00-05`.
The empty `ignored-attribute` does not prevent this: it is outside the decoration path.
Changing it or adding another named attribute does not change these bytes.
Likewise, in the question's example with `5 > @`, the result comes from `5`,
not from an encoding of the attributes of `five`.

This also explains why having no `@` is not, by itself, a problem:
a bytes object already has data, and an atom may produce an object with data.
An ordinary object with only named attributes has neither of these routes.

### An Object With Only Fields

```eo
[] > obj
  1 > one
  2 > two
  3 > three

(dataized obj).as-bytes > result
```

Forcing `result` fails in EO 0.36.0 because `obj` has no attached data,
no atom implementation, and no `@`.
It produces neither a concatenation of `1`, `2`, and `3` nor an empty byte sequence.
The field names and values are not automatically serialized.
Merely naming the expression `result` does not force its evaluation.

### Decorating an Object Without Data

```eo
[] > obj
  1 > one

[] > wrapper
  obj > @

(dataized wrapper).as-bytes > result
```

Forcing this `result` also fails.
The runtime follows `wrapper` to `obj`, where the same missing-data case occurs.
Adding a decoration layer does not give the object data.
By contrast, a chain of decorators that eventually reaches bytes can be dataized.

### A Decoration That Never Reaches Data

An endless decoration chain also produces no byte result.
For example, this recursive formation continually supplies another decorated object:

```eo
[] > forever
  forever > @

(dataized forever).as-bytes > result
```

There is no data at the end of this path, because there is no end.
The recursive runtime procedure cannot finish with bytes;
resource exhaustion may stop it instead.
Do not depend on a particular stack limit, exception message, or timeout.
Dataization does not turn a cycle into a reference identifier,
as a serializer with an explicit cycle-handling format might do.

### What the Wrapper Does

The [`dataized` atom implementation][implementation] dataizes its target
and wraps the resulting bytes in a new bytes object.
It does not inspect field names, define a wire format, or promise to reconstruct
the original object from those bytes.
Different objects can therefore have identical dataization results while exposing
different attributes.

To serialize a record, choose an explicit format and define how each field is encoded.
An object that implements that encoding can itself be dataized to obtain its output bytes.
That is an application-specific encoding, rather than a service supplied by `dataized`.

[question]: https://github.com/objectionary/news.eolang.org/issues/73
[dataized]: https://github.com/objectionary/eo/blob/0.36.0/eo-runtime/src/main/eo/org/eolang/dataized.eo
[delta]: https://github.com/objectionary/eo/blob/0.36.0/eo-runtime/src/main/java/org/eolang/PhDefault.java#L288-L303
[assets]: https://news.eolang.org/2024-05-14-rho-sigma-delta-lambda.html
[implementation]: https://github.com/objectionary/eo/blob/0.36.0/eo-runtime/src/main/java/EOorg/EOeolang/EOdataized.java#L61-L67
