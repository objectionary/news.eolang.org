---
layout: post
date: 2023-12-08
title: "Convert EO to φ-calculus Expression and Back"
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

Its [XMIR](https://news.eolang.org/2022-11-25-xmir-guide.html) representation (default location is 
`target/eo/2-optimize`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<program dob="2023-03-19T00:00:00"
         ms="12"
         name="foo.x.main"
         revision="1234567"
         source="/var/folders/xj/f4s949893tq37zzw0pdmd68h0000gn/T/junit2727451398454004374/foo/x/main.eo"
         time="2023-12-13T10:04:53.022835Z"
         version="0.0-SNAPSHOT">
   <!--This is XMIR - a dialect of XML, which is used to present a parsed EO program. For more information please visit https://news.eolang.org/2022-11-25-xmir-guide.html-->
   <listing>+package eo.example

[n] &gt; fibonacci
  if. &gt; @
    lt.
      n
      2
    n
    plus.
      fibonacci
        n.minus 1
      fibonacci
        n.minus 2
</listing>
   <errors>
      <error check="missing-home"
             line=""
             severity="warning"
             sheet="mandatory-home-meta"
             step="0">Missing home</error>
      <error check="missing-version-meta"
             line=""
             severity="warning"
             sheet="mandatory-version-meta"
             step="0">Missing version meta</error>
   </errors>
   <sheets>
      <sheet>not-empty-atoms</sheet>
      <sheet>duplicate-names</sheet>
      <sheet>many-free-attributes</sheet>
      <sheet>broken-aliases</sheet>
      <sheet>duplicate-aliases</sheet>
      <sheet>global-nonames</sheet>
      <sheet>same-line-names</sheet>
      <sheet>self-naming</sheet>
      <sheet>cti-adds-errors</sheet>
      <sheet>add-refs</sheet>
      <sheet>wrap-method-calls</sheet>
      <sheet>expand-qqs</sheet>
      <sheet>add-probes</sheet>
      <sheet>vars-float-up</sheet>
      <sheet>add-refs</sheet>
      <sheet>sparse-decoration</sheet>
      <sheet>unsorted-metas</sheet>
      <sheet>incorrect-architect</sheet>
      <sheet>incorrect-home</sheet>
      <sheet>incorrect-version</sheet>
      <sheet>expand-aliases</sheet>
      <sheet>resolve-aliases</sheet>
      <sheet>add-refs</sheet>
      <sheet>add-default-package</sheet>
      <sheet>broken-refs</sheet>
      <sheet>unknown-names</sheet>
      <sheet>noname-attributes</sheet>
      <sheet>duplicate-names</sheet>
      <sheet>duplicate-metas</sheet>
      <sheet>mandatory-package-meta</sheet>
      <sheet>mandatory-home-meta</sheet>
      <sheet>mandatory-version-meta</sheet>
      <sheet>correct-package-meta</sheet>
      <sheet>prohibited-package</sheet>
      <sheet>external-weak-typed-atoms</sheet>
      <sheet>unused-aliases</sheet>
      <sheet>unit-test-without-phi</sheet>
      <sheet>explicit-data</sheet>
      <sheet>set-locators</sheet>
   </sheets>
   <license/>
   <metas>
      <meta line="1">
         <head>package</head>
         <tail>eo.example</tail>
         <part>eo.example</part>
      </meta>
      <meta line="4">
         <head>probe</head>
         <tail>n.lt.if</tail>
         <part>n.lt.if</part>
      </meta>
   </metas>
   <objects>
      <o abstract=""
         line="3"
         loc="Φ.eo.example.fibonacci"
         name="fibonacci"
         pos="0">
         <o line="3" loc="Φ.eo.example.fibonacci.n" name="n" pos="1"/>
         <o base=".if"
            line="4"
            loc="Φ.eo.example.fibonacci.φ"
            name="@"
            pos="2">
            <o base=".lt" line="5" loc="Φ.eo.example.fibonacci.φ.ρ" pos="4">
               <o base="n"
                  line="6"
                  loc="Φ.eo.example.fibonacci.φ.ρ.ρ"
                  pos="6"
                  ref="3"/>
               <o base="org.eolang.int"
                  line="7"
                  loc="Φ.eo.example.fibonacci.φ.ρ.α0"
                  pos="6">
                  <o base="org.eolang.bytes"
                     data="bytes"
                     loc="Φ.eo.example.fibonacci.φ.ρ.α0.α0">00 00 00 00 00 00 00 02</o>
               </o>
            </o>
            <o base="n"
               line="8"
               loc="Φ.eo.example.fibonacci.φ.α0"
               pos="4"
               ref="3"/>
            <o base=".plus" line="9" loc="Φ.eo.example.fibonacci.φ.α1" pos="4">
               <o base="fibonacci"
                  line="10"
                  loc="Φ.eo.example.fibonacci.φ.α1.ρ"
                  pos="6"
                  ref="3">
                  <o base=".minus"
                     line="11"
                     loc="Φ.eo.example.fibonacci.φ.α1.ρ.α0"
                     pos="9">
                     <o base="n"
                        line="11"
                        loc="Φ.eo.example.fibonacci.φ.α1.ρ.α0.ρ"
                        pos="8"
                        ref="3"/>
                     <o base="org.eolang.int"
                        line="11"
                        loc="Φ.eo.example.fibonacci.φ.α1.ρ.α0.α0"
                        pos="16">
                        <o base="org.eolang.bytes"
                           data="bytes"
                           loc="Φ.eo.example.fibonacci.φ.α1.ρ.α0.α0.α0">00 00 00 00 00 00 00 01</o>
                     </o>
                  </o>
               </o>
               <o base="fibonacci"
                  line="12"
                  loc="Φ.eo.example.fibonacci.φ.α1.α0"
                  pos="6"
                  ref="3">
                  <o base=".minus"
                     line="13"
                     loc="Φ.eo.example.fibonacci.φ.α1.α0.α0"
                     pos="9">
                     <o base="n"
                        line="13"
                        loc="Φ.eo.example.fibonacci.φ.α1.α0.α0.ρ"
                        pos="8"
                        ref="3"/>
                     <o base="org.eolang.int"
                        line="13"
                        loc="Φ.eo.example.fibonacci.φ.α1.α0.α0.α0"
                        pos="16">
                        <o base="org.eolang.bytes"
                           data="bytes"
                           loc="Φ.eo.example.fibonacci.φ.α1.α0.α0.α0.α0">00 00 00 00 00 00 00 02</o>
                     </o>
                  </o>
               </o>
            </o>
         </o>
      </o>
   </objects>
</program>
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