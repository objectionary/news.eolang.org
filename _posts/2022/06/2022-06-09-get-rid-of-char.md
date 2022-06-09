---
layout: post
date: 2022-06-09
title: "Get rid of char object"
author: Graur
description: |
  Since `0.23.6` version of EO we get rid of char object at all.
  Now, if you have to use character, you can just use `string` object instead.
keywords:
  - eolang
  - updates
  - 0.23.6
---

Since `0.23.6` version of [EO](https://www.eolang.org) we get rid of `char` object.
Цe want to simplify core of the language and believe that this object can be easily replaced 
with its actual representation, i.e. an array of bytesю
Now, if you want to get the character, you can just use `string` object instead.
All of these examples are valid strings: `"\u0123"`, `"h"`, `"\t"`, `"\n"` and `"\07"`.
We get rid of `string.char-at` and `bytes.as-char` objects.

<!--more-->

If you need to get a character from the string use converting to a byte array instead: `"hello".as-bytes.part 0 1"` (it will return `"h"`).
And use `bytes.as-string` instead of `bytes.as-char`.

Please visit our [paper](https://arxiv.org/abs/2206.02585) to get more information.