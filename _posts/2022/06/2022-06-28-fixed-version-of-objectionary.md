---
layout: post
date: 2022-06-28
title: "How to Compile Against a Fixed Version of Objectionary"
author: yegor256
---

When you compile your EO code, the compiler discovers which objects
are "foreign" and tries to find them in Objectionary. It finds them,
downloads, and then compiles locally. The problem is that the objects
in Objectionary are not static: they get new versions very often. To
stabilize your build you may want to use fixed versions of objects.

Here is how, in your `pom.xml` (`hash` must include Git commit hash
from [`objectionary/home`](https://github.com/objectionary/home)):

```
<pom>
  <build>
    <plugins>
      <plugin>
        <groupId>org.eolang</groupId>
        <artifactId>eo-maven-plugin</artifactId>
        <configuration>
          <hash>0d94362</hash>
        </configuration>
      </plugin>
    </plugins>
  </build>

```

If you use [`eoc`](https://github.com/objectionary/eoc),
you can do it with the command line option:

```
$ eoc --hash=0d94362 compile
```

Full list of Git hashes of Objectionary is
[here](https://github.com/objectionary/home/commits/master).
