---
layout: post
date: 2026-11-07
title: "Void Attributes and NULL"
author: yegor256
---

A void attribute declares a place for an argument.
It lets us form an object before supplying all the objects it needs.
That can look like a Java reference initialized to `null`,
  which prompted [this discussion][discussion].
The difference becomes clear when we apply an argument:
  EO copies the object and fills a vacant attribute in the copy.

<!--more-->

### A Place for an Argument

Consider this formation and two applications:

```eo
[x] > box
  x > value
box 42 > answer
box 7 > lucky
```

The brackets declare `x` as void.
The attribute `value` refers to whatever object will be supplied as `x`.
Application supplies `42` in `answer` and `7` in `lucky`;
  `box` remains available for other applications.
We have not changed `answer` from one value to another.
We have made two objects from the same formation.

In the [runtime implementation][application], application first copies
  the target and then attaches the arguments to that copy.
Each [void attribute][void] accepts an object only once;
  trying to attach another object to an already filled attribute is rejected.
Copying a filled attribute preserves its binding rather than making it vacant.

### What Java's NULL Means

In Java, a reference can hold the special value `null`:

```java
Object value = null;
value = new Object();
value = null;
```

These assignments replace the contents of the same variable.
Java also permits passing `null` as an argument and testing a reference
  with `value == null`.
Declaring a Java reference `final` prevents reassignment,
  but still permits initializing it to `null`.
Thus, immutability alone does not explain the distinction.

In EO, the void declaration marks an argument that has not been supplied.
It does not attach a special object called NULL.
Supplying an argument fills that vacancy; it does not replace a NULL object.
The Java runtime uses Java `null` internally to record vacancy,
  but its attribute API rejects Java `null` as an argument.
That implementation detail does not introduce a NULL literal into EO.

### Reading an Unfilled Attribute

Leaving an argument unfilled does not guarantee that every use will succeed.
In the [original discussion][old-behavior], taking an unfilled attribute
  produced a runtime error.
The language has since changed: as explained in [The Terminator][terminator],
  taking an unfilled void now yields bottom, written `T` in EO.
The linked [runtime snapshot][void] implements this behavior
  by returning a `PhTerminator` when the attribute is vacant.
Forcing bottom through dataization terminates the computation.

This matters when adapting old examples: do not promise an immediate
  exception just because an argument was omitted.
A void declaration, the object later supplied to it,
  and the bottom produced by reading it while vacant are distinct things.

### Empty Bytes Are Supplied Data

The same discussion also asked whether empty bytes were NULL.
They are not: `--` denotes a byte sequence containing no bytes,
  while `00-` contains one zero byte.
The Java counterparts are `new byte[0]` and `new byte[] {0}`;
  both are arrays, and neither reference is `null`.

We can supply either sequence as an argument:

```eo
box -- > empty
box 00- > zero
```

In both applications `x` has been filled.
An empty sequence is still supplied data;
  it does not leave a vacancy in the object.
Use a void attribute to declare an input, and apply an object to supply it.
Use empty bytes when the input really is a sequence of length zero.

[discussion]: https://github.com/objectionary/eo/issues/3545
[old-behavior]: https://github.com/objectionary/eo/issues/3545#issuecomment-2497272169
[application]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/java/org/eolang/PhApplication.java#L53-L60
[void]: https://github.com/objectionary/eo/blob/94d598975a45bb3089a85c4d6bc19c5a0b0343b1/eo-runtime/src/main/java/org/eolang/AtVoid.java
[terminator]: https://news.eolang.org/2026-07-31-the-terminator.html
