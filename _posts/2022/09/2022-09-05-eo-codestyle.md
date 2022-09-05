---
layout: post
date: 2022-09-05
title: "Codestyle checks in EO"
author: McJohn974
---

One of the goals of EOlang was clear and readable code. Thats why it has some codestyle checks processed by the compiler.

Firstly, level of nesting is controlled by the indentations. 
Reason -- code with different indents is hard to read and understand,
and code with correct indents is clear without parenesses. Size of indent is strict -- two spaces. 
It was done for big decoration hierarchy. Also, EOlang strictly controls all space symbols -- no
more than one whitespace between words, no more than one blank line between objects and attributes, 
no blank lines in the code of one attribute, no tabs. If code looks ugly without extra blank lines,
it is a good marker that you should create new object or attribute.


EOlang control not only indentations, but also naming. You can't create an object or a
attribute with first capital letter.

EO compiler checks all useless includes, and force programmer to remove them.