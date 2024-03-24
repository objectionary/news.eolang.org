---
layout: post
date: 2024-03-18
title: "Recursion in cage"
author: levBagryansky
---

The main difference between `cage` and `memory` is how the object is written.
When dataizing, `memory.write` datazies its argument as well. Whereas 
`cage.write` just saves a reference to this object. As a result, the dataization
of the `cage` may turn out to be recursive. Unfortunately, this error is often not obvious.

In this post we will tell you how we solved this problem.

<!--more-->

# A little about `cage`

In [this](https://news.eolang.org/2023-08-04-storing-objects-formed-differently-into-cage.html) post we 
raised the topic of `cage`. There we told how to write into `cage`, and what limitations there are -
objects of the same `forma` can be written into one cage.
Here is a simple example of using a `cage`:

```eo
# Cage conatains int.
[] > cage-contains-int
  cage 0 > my-int
  eq. > @
    seq
      *
        my-int.write 42
        my-int
    42
```

But you might also accidentally write `cage` inside `cage`, leading to infinite recursion.
```eo
[] > catches-stack-overflow
  cage > cge
    0.plus
  seq > @
    *
      cge.write
        0.plus cge
      cge
```
In order to provide everywhere the same `forma` we use `0.plus`.
When `cge` is dataizing it dataizes `0.plus cge`. It takes attribute `plus` from object `0`. `int.plus`
tries to dataize its argument `cge` inside. We have achieved our goal: infinite recursion is obtained.
The first attempt is to detect such cases even in compilation. But generally speaking, the presence 
of `cage` inside `cage` is not a mistake, because such a program may well be correct.
Dataization of `cage` can several times pas through the `cage` and only then return some value.
This eo illustrates such scenario (it does not work with latest version but idea of deep `cage` is clear):
```eo
# correct-cage-recursion.
[] > correct-cage-recursion
  cage > cge
    if.
      i.as-bool
      TRUE
      seq
        *
          i.write
            TRUE
          $.cge
  memory > i
    FALSE
  cge > @

```
Therefore, it is necessary to detect recursion in runtime somehow. Until recently `EOcage::lambda`
was the following:
```java
@Override
public Phi lambda() {
    return this.attr("enclosure").get();
}
```
That is, when dataizing, `cage` simply returned its `enclosure`. In order to trace how many times 
`cage` dataization uses itself we wrap `enclosure` in the `PhTracedEnclosure`:
```java
@Override
public Phi lambda() {
    return new PhTracedEnclosure(this.attr("enclosure").get(), this);
}
```
`PhTracedEnclosure` keeps `enclosure` and `cage` that it traces. It also has something like **static** 
`Map<Phi, Integer>`, where it counts how many times `enclosure` was used: It increments `cage`'s 
counter before calling `enclosure::attr` method and decrement after. Thus, in the case of recursion, 
the counter does not decrease, but only increases. When the counter reaches a certain threshold, an
exception `ExFailure` is thrown. 

The value of such a threshold is called `"EO_MAX_CAGE_RECURSION_DEPTH"`, and it is a java property 
which you can set by yourself. We chose `100` to be default value. Then `cage` can be
recursive enough, but it will detect problem before throwing of `StackOverflowError`. You can 
set environment  variable via `-D` flag.

Keep exploring EO, and remember, there's always more to discover. Stay connected for future updates, 
and We look forward to catching up with you soon!
