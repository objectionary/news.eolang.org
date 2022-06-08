---
layout: post
date: 2022-06-08
title: "Are you still using chars? We don't!"
author: Graur
description: |
  Since `0.23.6` version of EO we get rid of char object at all.
  Now, if you have to use character, you can just use `string` object instead.
keywords:
  - eolang
  - oop
  - object-oriented programming
---

Since `0.23.6` version of [EO](https://www.eolang.org) we get rid of `char` object at all.
Now, if you want to get the character, you can just use `string` object instead.
All of these examples are valid strings: `"\u0123"`, `"h"`, `"\t"`, `"\n"` and `"\07"`.
Also, you can get any symbol from any string like this: `"hello".as-bytes.part 0 1"`. It returns `"h"`.

<!--more-->

Please visit our [paper](https://arxiv.org/abs/2206.02585) to get more information.