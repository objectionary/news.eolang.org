---
layout: post
date: 2022-10-19
title: "EO Build Process: Placed Catalog"
author: mximp
---

As described in [this blog post](https://www.yegor256.com/2021/10/21/objectionary.html#place-)
`place` step of EO build process copies compiled files from dependencies' JARs
into `target/classes` folder to have them in classpath during compilation later.
You can check [`PlaceMojo`](https://github.com/objectionary/eo/blob/master/eo-maven-plugin/src/main/java/org/eolang/maven/Place.java)
class for exact logic of this step.
All copied files are registered within "placed catalog," which is normally stored as
`target/eo/placed.csv` file.
Let's consider structure of the catalog in more details.

<!--more-->

Each entry within the catalog corresponds to a single file that has been copied.

There are two kinds of entries. Entries of kind `class` corresponds to a file within JAR.
It can be `.class` or `.xml` or any other type of file which needs to be within classpath
during compilation.
Another kind, `jar`, represents dependency JAR file whose content has been copied.

Placed catalog stores the following attributes for each entry:

* `id`:
For `class`-kind it is absolute path to a copied file in target location.
For `jar`-kind it is a JAR dependency reference constructed from Maven artifacts
(e.g. `org.eolang/eo-strings/-/0.0.4`).

* `related`:
For `class`-kind relative path to a file within classpath.
Empty for `jar` kind.

* `dependency`:
For `class`-kind dependency which the file was extracted from.
Empty for `jar` kind.

* `kind`:
Type of the entry (see description above).

* `hash`:
For `class`-kind stores MD5 hash of the file content.
Empty for `jar` kind.

* `unplaced`:
Contains `true` if the file was removed during `unplace` step
(see [`UnplaceMojo`](https://github.com/objectionary/eo/blob/master/eo-maven-plugin/src/main/java/org/eolang/maven/Unplace.java)).
Otherwise, it is set to `false`.

The information from "placed catalog" can be useful when investigating
"Class not found"-like issues during compilation. It can show what was added/removed
from build's classpath.
It also shows the origins of classes.
