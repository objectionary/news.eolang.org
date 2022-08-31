---
layout: post
date: 2022-08-30
title: "Parser of EO as an External Java Library"
author: yegor256
---

EO parser is written in [ANTLR4](https://www.antlr.org),
Java, and [XSL](https://en.wikipedia.org/wiki/XSL).
It is packaged as a multi-module [Maven](https://maven.apache.org) project, in
[objectionary/eo](https://github.com/objectionary/eo) GitHub repository.
In order to compile an EO program to Java you may either use
our [Maven plugin](https://github.com/objectionary/eo/tree/master/eo-maven-plugin)
or our [eoc](https://github.com/objectionary/eoc) command line toolkit.
Moreover, you can also use our parser programmatically in order to generate XMIR from EO.
Here is how you do it in your Java/Kotlin/Clojure/Groovy/etc. project.

<!--more-->

First you need to put this dependency into `compile` scope (make sure you
use the [latest](https://github.com/objectionary/eo/releases) version):

```xml
<dependency>
  <groupId>org.eolang</groupId>
  <artifactId>eo-parser</artifactId>
  <version>0.27.2</version>
</dependency>
```

Then, you need [Xsline](https://github.com/yegor256/xsline/) too (again, check the latest
version [here](https://github.com/yegor256/xsline)):

```xml
<dependency>
  <groupId>com.yegor256</groupId>
  <artifactId>xsline</artifactId>
  <version>0.11.0</version>
</dependency>
```

Finally, you need [Cactoos](https://github.com/yegor256/cactoos/) and
[jcabi-xml](https://github.com/jcabi/jcabi-xml):

```xml
<dependency>
  <groupId>org.cactoos</groupId>
  <artifactId>cactoos</artifactId>
  <version>0.54.0</version>
</dependency>
<dependency>
  <groupId>com.jcabi</groupId>
  <artifactId>jcabi-xml</artifactId>
  <version>0.25.3</version>
</dependency>
```

Now, you are ready to parse a EO program to XMIR:

```java
import java.io.ByteArrayOutputStream;
import org.eolang.parser.Syntax;
import org.cactoos.io.InputOf;
import org.cactoos.io.OutputTo;
import com.jcabi.xml.XML;

ByteArrayOutputStream baos = new ByteArrayOutputStream();
new Syntax(
  "foo", // just a name to be put into XMIR
  new InputOf(Paths.get("test/foo.eo")),
  new OutputTo(baos)
).parse();
XML xmir = new XMLDocument(baos.toByteArray());
```

You can also parse it from a `String`: just give it to the
constructor of `InputOf`.

Then, you can send this XMIR through a default "train"
of optimizing XSL stylesheets:

```java
import com.jcabi.xml.XML;
import com.yegor256.xsline.Xsline;
import org.eolang.parser.ParsingTrain;

XML after = new Xsline(new ParsingTrain()).pass(xmir);
```

If you want to add a few extra XSL stylesheets to the train,
here is how:

```java
XML after = new Xsline(
  new TrClasspath<>(
    new ParsingTrain()
    "/org/eolang/parser/add-refs.xsl",
    "/org/foo/simple.xsl"
  ).back()
).pass(xmir);
```

Here, `/org/eolang/parser/add-refs.xsl` is the XSL file available in classpath
thanks to the `eo-parser.jar`, while `/org/foo/simple.xsl` is the XSL you
have in your own project in `src/main/resources/org/foo/simple.xml`.

If, instead of adding XSL stylesheets to the default collection,
you want to use your own set of XSL stylesheets, do this
(mind the usage of `empty()`):

```java
XML after = new Xsline(
  new TrClasspath<>(
    new ParsingTrain().empty(),
    "/org/eolang/parser/add-refs.xsl",
    "/org/foo/simple.xsl"
  ).back()
).pass(xmir);
```

That's it :)
