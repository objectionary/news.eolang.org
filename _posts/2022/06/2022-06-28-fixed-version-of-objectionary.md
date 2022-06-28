---
layout: post
date: 2022-06-28
title: "How to Compile Against a Fixed Version of Objectionary"
author: yegor256
---

When you compile your EO code, the compiler discovers which objects
are "foreign" and tries to find them in [Objectionary](https://github.com/objectionary/home). It finds them,
downloads, and then compiles locally. The problem is that the objects
in Objectionary are not static: they get new versions very often.
That's why, in order to stabilize your build you may want
to use their fixed versions.

<!--more-->

Here is how, in your `pom.xml` (`hash` must include Git commit hash
from [`objectionary/home`](https://github.com/objectionary/home)):

```
<project>
  [...]
  <build>
    [...]
    <plugins>
      [...]
      <plugin>
        <groupId>org.eolang</groupId>
        <artifactId>eo-maven-plugin</artifactId>
        <configuration>
          <hash>0d94362</hash>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

If you use [`eoc`](https://github.com/objectionary/eoc),
you can do it with the command line option:

```
$ eoc --hash=0d94362 compile
```

Full list of Git hashes of Objectionary is
[here](https://github.com/objectionary/home/commits/master).
