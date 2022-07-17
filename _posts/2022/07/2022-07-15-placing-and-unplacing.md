---
layout: post
date: 2022-07-15
title: "Placing and Unplacing in JAR Artifacts"
author: yegor256
---

The entire process of packaging EO objects and atoms into JAR artifacts
is explained in this blog post:
[Objectionary: Dictionary and Factory for EO Objects](https://www.yegor256.com/2021/10/21/objectionary.html).
It's pretty straight forward. However, there is one tricky situation
related to placing compiled Java binaries into the JAR. This process
may go wrong and sometimes it does. In version 0.24.0 of
[our Maven plugin](https://github.com/objectionary/eo/tree/master/eo-maven-plugin) we
introduced a pair of options for the `unplace` goal. Here is how they work.

<!--more-->

First, when you build a JAR artifact in your Maven project, you have some `.eo`
sources in `src/main/eo` and some Java sources in `src/main/java`. They both
are compiled into `.class` files into `target/classes`.

Second, in order for your program to work, it needs `.eo` files for objects,
which it finds in [Objectionary](https://github.com/objectionary/home). The files
are "pulled" and then saved into `target/eo/04-pull`. Then, they also
are compiled into `.class` binaries and also _placed_ into `target/classes`,
mixing together with your files.

Third, your program needs `.class` binaries from inside JAR artifacts,
which some `.eo` objects point to by means of `+rt` meta. The artifacts
are downloaded, unpacked, and then placed into `target/classes`.

Finally, it's time to package your own JAR and release it to Maven Central.
Obviously, we don't want all the files. previously placed into `target/classes`,
to be packaged into the JAR. We only want those that were compiled from _your_
source `.java` files. We don't even want those `.class` files that were
compiled from the auto-generated `.java` from your `.eo` objects.
We only want compiled _atoms_ to be in the JAR.

The Maven plugin goal `unplace` cleans up the `target/classes` directory and
removes unnecessary binaries, which were placed into it earlier. It understands
which files to remove, thanks to the catalog it maintains during the entire
build cycle, in `target/eo/placed.csv` file. The goal `unspile` adds more
cleaning by removing `.class` files generated from your `.java` classes.

However, sometimes they may make mistakes. For example, when a JAR artifact
coming from Maven Central and your own Java files have the same classes. The plugin
won't understand which of them to keep and most probably will delete both.

To make its behavior fully explicit, you may use one of these two options (or both):

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
        <executions>
          <execution>
            <id>compile</id>
            <goals>
              <goal>register</goal>
              <goal>assemble</goal>
              <goal>transpile</goal>
              <goal>copy</goal>
              <goal>unplace</goal>
              <goal>unspile</goal>
            </goals>
            <configuration>
              <keepBinaries>
                <glob>EOorg/EOeolang/EOfoo/**</glob>
              </keepBinaries>
              <removeBinaries>
                <glob>EOorg/EOeolang/**.class</glob>
              </removeBinaries>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

This configuration will ensure that only `EOorg/EOeolang/EOfoo/**` files
will stay in `target/classes` before the JAR is packaged. Also, if for
some magic reason `EOorg/EOeolang/**.class` will remain their, they will
also be deleted.

I think it's a good practice to use `keepBinaries` option in your
library, just to be safe and sure that nothing aside from your compiled
atoms get into the JAR.




