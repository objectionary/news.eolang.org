---
layout: post
date: 2022-11-25
title: "XMIR, a Quick Tour"
author: yegor256
---

_Last updated at: 17.04.2025_

XMIR is a dialect of [XML](https://en.wikipedia.org/wiki/XML),
which we use to represent a parsed
[EO](https://www.eolang.org) object. It is a pretty simple format,
which has a few
important tricks, which I share below in this blog post. You may
also want to check our [schema](https://en.wikipedia.org/wiki/XML_schema):
[`XMIR.xsd`][xsd]
(it is also [rendered in HTML](https://www.eolang.org/XMIR.html),
which may be more readable for some of you).

<!--more-->

Consider this simple EO object that prints `"Hello, world!"`:

```
# App.
[] > app
  [x] > foo
    QQ.io.stdout > @
      QQ.txt.sprintf *1
        "Hello, %s\n"
        x
  foo > @
    "world!"
```

If we parse it using `EoSyntax` class from [eo-parser],
we will get this XMIR (or very similar):

```xml
<object 
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 dob="2024-12-27T11:00:08" 
 ms="98" 
 revision="27abe8b"
 time="2025-04-17T09:32:04.455112Z" 
 version="0.56.0"
 xsi:noNamespaceSchemaLocation="https://www.eolang.org/xsd/XMIR-0.56.0.xsd">
 <listing># App.
[] &gt; app
  [x] &gt; foo
    QQ.io.stdout &gt; @
      QQ.txt.sprintf *1
        "Hello, %s\n"
        x
  foo &gt; @
    "world!"
</listing>
  <o line="2" name="app" pos="0">
    <o line="3" name="foo" pos="2">
      <o base="∅" line="3" name="x" pos="3"/>
      <o base=".stdout" line="4" name="@" pos="9">
        <o base=".io" line="4" pos="6">
          <o base="QQ" line="4" pos="4"/>
        </o>
        <o base=".sprintf" line="5" pos="12">
          <o base=".txt" line="5" pos="8">
            <o base="QQ" line="5" pos="6"/>
          </o>
          <o base="string" line="6" pos="8">48-65-6C-6C-6F-2C-20-25-73-0A</o>
          <o base="tuple" line="7" pos="8">
            <o base=".empty">
              <o base="tuple"/>
            </o>
            <o base="x" line="7" pos="10"/>
          </o>
        </o>
      </o>
    </o>
    <o base="foo" line="8" name="@" pos="2">
      <o base="string" line="9" pos="4">77-6F-72-6C-64-21</o>
    </o>
  </o>
</object>
```

The `<object>` is the root element, it will always be there, with
a few mandatory attributes:

* `ms` is how much time in milliseconds it took to parse the object
and generate this XMIR file,
* `time` is the time in [ISO 8601] format when the file was generated,
* `version` is the version of the parser.

The `<listing>` element contains the source code of the EO object,
which was parsed, without any modifications, "as is."

## Errors and Warnings

The `<errors>` element may have a list of problems discovered by the
parser or any other optimizers, as `<error>` elements. If there are no
errors, the `<errors>` element should not exist in `<object>`.
For example, it may look like this:

```xml
<object>
  [...]
  <errors>
    <error severity="warning" line="3">There is an extra bracket</error>
    <error severity="error" line="12">The object 'x' is not found</error>
    [...]
  </errors>
</object>
```

The errors with the `warning` severity may more or less safely be ignored. The
errors with the `error` severity will lead to failures in further compilation
and processing. There could also be elements with the `critical` severity,
which must stop the processing of the document immediately.

## Sheets

The `<sheets>` element contains a list of all
post-processors that were applied to the document after is parsing.
We process our XMIR documents using dozens of XSL stylesheets. That's why
the name of the XML element. You may find something like this over there:

```xml
<object>
  [...]
  <sheets>
    <sheet>move-voids-up</sheet>
    <sheet>const-to-dataized</sheet>
    <sheet>stars-to-tuples</sheet>
    <sheet>wrap-method-calls</sheet>
    [...]
  </sheets>
</object>
```

The names you see in the `<sheet>` elements are the names of the files.
For example, `wrap-method-calls` represents the
[`wrap-method-calls.xsl`] file
in the [objectionary/eo](https://github.com/objectionary/eo) GitHub repository.

If no XSL stylesheets are applied to XMIR, the `<sheets>` element should not exist
in `<object>`.

## Metas

There may be an optional element `<metas>` with a list of `<meta>` elements.
For example, if my source code would have this meta at the 3rd
line of the source file:

```
+alias foo com.example.foo
```

We would see the following in the XMIR:

```xml
<object>
  [...]
  <metas>
    <meta line="3">
      <head>alias</head>
      <tail>foo Q.com.example.foo</tail>
      <part>foo</part>
      <part>Q.com.example.foo</part>
    </meta>
    [...]
  </metas>
</object>
```

Each `<meta>` element contains parts of the meta. The `<head>`
contains everything that goes after the `+` until the first space.
The `<tail>` contains everything after the first space. There could
be a number of `<part>` elements, each of which containing the parts
of the `<tail>` separated by spaces.

## Objects

The `<object>` element must contain only one `<o/>` element which represents an 
object being parsed. The `<o/>` element may have a few optional attributes:

* `line` and `pos` are the number of the line where the object
was found by the parser and the position in the line;
* `name` is the name of the object, if the object has it;
* `base` may refer to object formation that is being copied;
* `as` is the name of the attribute which current object is bound to during the
application

There could be no other attributes.

## Special cases

1. The `<o/>` elements that have nested `<o>` element with `name` which 
value is `λ` are **atoms**. Atoms must not have `base` attribute:
```xml
<o name="try">
  <o name="λ"/>
</o>
```

2. The `<o/>` elements with `base` attribute which value is `∅` are **void** attributes.
Void attributes also must have `name` attribute:
```xml
<o name="foo">
  <o name="bar" base="∅"/>
</o>
```

3. **Data literals** found in the source code are presented with nested `<o/>` XML elements
that contain text. Only elements with `base` attribute equal to `Q.org.eolang.bytes` may contain
nested `<o>` element with text.

```xml
<o base="Q.org.eolang.bytes" line="6" pos="8">
  <o>48-65-6C-6C-6F-2C-20-25-73-0A</o>
</o>
```

4. The `name` attribute of `<o/>` element may be **auto generated** by EO parser. 
In such case it's look like:
```xml
<o name="a🌵104"/>
```

Such `name` consists of several parts:
- char `a` (ascii 97) that stands for "auto-generated"
- char `🌵` that is just a pretty character prohibited by EO grammar
- number `104` which is joined line and position of the place where 
the object is found

5. If object is bound to a specific attribute not by name but by position, the
`as` attribute may look like:
```xml
<o base="Q.org.eolang.number" as="α2"/>
```
Here the first character is `α` (alpha), the number `2` is the position of the 
attribute.

<hr/>

This description of XMIR is not complete. If you want me to explain
something else in more details, please post a message below this blog post
and I will add the content.

[xsd]: https://raw.githubusercontent.com/objectionary/eo/gh-pages/XMIR.xsd
[eo-parser]: https://github.com/objectionary/eo/tree/master/eo-parser
[ISO 8601]: https://en.wikipedia.org/wiki/ISO_8601
[`not-empty-atoms.xsl`]: https://github.com/objectionary/eo/blob/master/eo-parser/src/main/resources/org/eolang/parser/errors/not-empty-atoms.xsl
[`set-locators.xsl`]: https://github.com/objectionary/eo/blob/master/eo-parser/src/main/resources/org/eolang/parser/set-locators.xsl
