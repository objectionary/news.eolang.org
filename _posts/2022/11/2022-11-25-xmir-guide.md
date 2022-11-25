---
layout: post
date: 2022-11-25
title: "XMIR, a Quick Tour"
author: yegor256
---

XMIR is a dialect of [XML](https://en.wikipedia.org/wiki/XML), which we use to represent a parsed
  [EO](https://www.eolang.org) program. It is a pretty simple format, which has a few
  important tricks, which I share below in this blog post. You may
  also want to check our [schema](https://en.wikipedia.org/wiki/XML_schema):
  [`XMIR.xsd`](https://raw.githubusercontent.com/objectionary/eo/gh-pages/XMIR.xsd).

<!--more-->

Consider this simple EO program that prints `"Hello, world!"`:

```
[] > app
  [x] > foo
    QQ.io.stdout > @
      QQ.txt.sprintf
        "Hello, %s\n"
        x
  foo > @
    "world!"
```

If you parse it using `Syntax` class from eo-parser (without
any optimizations), you will get this XMIR:

```xml
<program ms="401" name="test-1" time="2022-11-25T11:50:45.419417Z" version="1.0-SNAPSHOT">
  <listing>[] &gt; app
  [x] &gt; foo
    QQ.io.stdout &gt; @
      QQ.txt.sprintf
        "Hello, %s\n"
        x
  foo &gt; @
    "world!"
</listing>
  <errors/>
  <sheets/>
  <objects>
    <o abstract="" line="1" name="app" pos="0">
      <o abstract="" line="2" name="foo" pos="2">
        <o line="2" name="x" pos="3"/>
        <o base="QQ" line="3" pos="4"/>
        <o base=".io" line="3" method="" pos="6"/>
        <o base=".stdout" line="3" method="" name="@" pos="9">
          <o base="QQ" line="4" pos="6"/>
          <o base=".txt" line="4" method="" pos="8"/>
          <o base=".sprintf" line="4" method="" pos="12">
            <o base="string" data="string" line="5" pos="8">Hello, %s\n</o>
            <o base="x" line="6" pos="8"/>
          </o>
        </o>
      </o>
      <o base="foo" line="7" name="@" pos="2">
        <o base="string" data="string" line="8" pos="4">world!</o>
      </o>
    </o>
  </objects>
</program>
```

The element `<program>` is the root element and it will always be there.
It has a few mandatory attributes:

  * `ms` is how much time in milliseconds it took to parse the program and generate this XMIR file,
  * `name` is the name of the program, as it was given to the parser,
  * `time` is the time in ISO 8601 format when the file was generated,
  * `version` is the version of the parser.

The `<listing>` element contains the source code of the EO program, which was parsed, without
any modifiations, "as is."

## Errors and Warnings

The `<errors>` element is always there, but it may either be empty, which means that
  the parser didn't find any errors, or may have a few `<error>` elements. Usually,
  such elements are added there by optimizers and other software that does post-processing
  of the XMIR file. For example, it may look like this:

```xml
<program>
  [..]
  <errors>
    <error severity="warning" line="3">There is an extra bracket</error>
    <error severity="error" line="12">The object 'x' is not found</error>
  </errors>
</program>
```

The errors with the `warning` severity may more or less safely be ignored. The
  errors with the `error` severity will lead to failures in further compilation
  and processing. There could also be elements with the `critical` severity,
  which must stop the processing of the document immediately.

## Sheets

The `<sheets>` element will rarely be empty. It contains a list of all
  post-processors that were applied to the document after is parsing.
  We process our XMIR documents using dozens of XSL stylesheets. That's why
  the name of the XML element. You may find something like this over there:

```xml
<program>
  [..]
  <sheets>
    <sheet>not-empty-atoms</sheet>
    <sheet>middle-varargs</sheet>
    <sheet>duplicate-names</sheet>
    <sheet>many-free-attributes</sheet>
    [...]
  </sheets>
</program>
```

The names you see in the `<sheet>` elements are the names of the files.
  For example, `not-empty-atoms` represents the
  [`not-empty-atoms.xsl`](https://github.com/objectionary/eo/blob/master/eo-parser/src/main/resources/org/eolang/parser/errors/not-empty-atoms.xsl) file
  in the [objectionary/eo](https://github.com/objectionary/eo) GitHub repository.

## Metas

There may be an optional element `<metas>` with a list of `<meta>` elements. For example,
  if my source code would have this meta:

```
+architect yegor256@gmail.com
```

I would see the following in my XMIR:

```xml
<program>
  [..]
   <metas>
    <meta line="23">
      <head>architect</head>
      <tail>yegor256@gmail.com</tail>
      <part>yegor256@gmail.com</part>
    </meta>
    [..]
  </metas>
</program>
```

Each `<meta>` element contains parts of the meta. The `<head>` contains everything that goes after
the `+` until the first space. The `<tail>` contains everything after the first space. There could
be a number of `<part>` elements, each of which containing the parts of the `<tail>` separated by spaces.

## Objects

The `<objects/>` element contains object, as they are found in the source
  code, where each objects is represented by the `<o>` element. Each `<o>` element
  may have a few optional attributes:

  * `line` and `pos` are the number of the line where the object was found by the parser and the position in the line (these two numbers uniquely identify the object in the source code);
  * `abstract` is the attribute that you will see only in abstract objects;
  * `name` is the name of the object, if the object has it;
  * `base` may refer to an abstract object that is being copied;
  * `method` may be present if the `base` starts with a dot;
  * `data` may be present if the object is the data object.

## Data Objects

Data literals found in the source code are presented with `<o>` XML elements
  that contain the data, for example:

```xml
<o base="string" data="string" line="8" pos="4">world!</o>
```

The value of the `data` attribute is the "type" of the data found in the
sources. It may be one of the following five:
`string`, `float`, `int`, `bool`, and `bytes`. For example:

```xml
<o data="string">Hello, 大家!</o>
<o data="float">3.141593</o>
<o data="int">65535</o>
<o data="bool">TRUE</o>
<o data="bytes">world!</o>
```

