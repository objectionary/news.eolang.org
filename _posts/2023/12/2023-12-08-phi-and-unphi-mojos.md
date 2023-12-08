---
layout: post
date: 2023-12-08
title: "Covert EO to φ-calculus expression and back"
author: maxonfjvipon
---

In the recently released version [0.34.0](https://github.com/objectionary/eo/releases/tag/0.34.0), 
we have implemented several changes to EO. Today, we will discuss the conversion of EO 
to φ-calculus expression and vice versa.

<!--more-->

As you may know, φ-calculus is a formal model that we are attempting to use as a base for
object-oriented programming languages, including EO.

From now on, it is possible to convert a program written in EO to a φ-calculus expression. 
Here's how you can accomplish this in your personal project.

First you need to add `eo-maven-plugin` dependency:
```xml
<dependency>
  <groupId>org.eolang</groupId>
  <artifactId>eo-maven-plugin</artifactId>
  <version>0.34.0</version> <!-- use 0.34.0 or younger -->
</dependency>
```

Conversion occurs in four stages: registration, parsing, optimization, and actual conversion. 
Therefore, include the corresponding goals in your build pipeline:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.eolang</groupId>
      <artifactId>eo-maven-plugin</artifactId>
      <version><!-- use the last version --></version>
      <executions>
        <execution>
          <id>convert</id>
          <goals>
            <goal>register</goal>
            <goal>parse</goal>
            <goal>optimize</goal>
            <goal>xmir-to-phi</goal>
          </goals>
        </execution>
        <configuration>
          <sourcesDir><!-- Where .eo source is placed, src/main/eo by default --></sourcesDir>
          <phiOutputDir><!-- Where .phi files are placed target/eo/phi by default --></phiOutputDir>
        </configuration>
      </executions>
    </plugin>
  </plugins>
</build>
```

Now, you are ready to convert an EO program to a φ-calculus expression. Place the `.eo` file in the source
directory and run your Maven project. When it's done, files with the `.phi` extension are placed in
the `phiOutputDir` directory.

You may also convert a φ-calculus expression back to EO. 
For this, your build pipeline should look like:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.eolang</groupId>
      <artifactId>eo-maven-plugin</artifactId>
      <version><!-- use the last version --></version>
      <executions>
        <execution>
          <id>convert</id>
          <goals>
            <goal>phi-to-xmir</goal>
            <goal>optimize</goal>
            <goal>print</goal>
          </goals>
        </execution>
        <configuration>
          <unphiInputDir><!-- Where .phi source is placed, target/eo/phi by default --></unphiInputDir>
          <printSourcesDir>${project.basedir}/target/eo/2-optimize</printSourcesDir> <!-- /src/main/xmir by default -->
          <printOutputDir><!-- Where result .eo files are placed, target/generated-sources/eo by default --></printOutputDir>
        </configuration>
      </executions>
    </plugin>
  </plugins>
</build>
```

When it's done result `.eo` files are placed in output `printOutputDir` directory.

Here's an example of how the EO Fibonacci program looks in φ-calculus (you can find more examples
[here](https://github.com/objectionary/eo/tree/master/eo-maven-plugin/src/test/resources/org/eolang/maven/phi)):
```eo
+package eo.example

[n] > fibonacci
  if. > @
    lt.
      n
      2
    n
    plus.
      fibonacci
        n.minus 1
      fibonacci
        n.minus 2
```

And its φ-calculus representation:

```phi
{
    eo ↦ ⟦
        example ↦ ⟦
            fibonacci ↦ ⟦
                n ↦ ∅, 
                φ ↦ ξ.n.lt(
                    α0 ↦ Φ.org.eolang.int(
                        α0 ↦ Φ.org.eolang.bytes(Δ ⤍ 00-00-00-00-00-00-00-02)
                    )
                ).if(
                    α0 ↦ ξ.n, 
                    α1 ↦ ξ.ρ.fibonacci(
                        α0 ↦ ξ.n.minus(
                            α0 ↦ Φ.org.eolang.int(
                                α0 ↦ Φ.org.eolang.bytes(Δ ⤍ 00-00-00-00-00-00-00-01)
                            )
                        )
                    ).plus(
                        α0 ↦ ξ.ρ.fibonacci(
                            α0 ↦ ξ.n.minus(
                                α0 ↦ Φ.org.eolang.int(
                                    α0 ↦ Φ.org.eolang.bytes(Δ ⤍ 00-00-00-00-00-00-00-02)
                                )
                            )
                        )
                    )
                )
            ⟧, 
            λ ⤍ Package
        ⟧, 
        λ ⤍ Package
    ⟧
}
```

That's it for now, be in touch.