---
layout: post
date: 2022-09-05
title: "EO codestyle"
author: MCJOHN974
---


One of the goals of EOlang was clear and readable code. 
That's why it has some codestyle checks processed by the compiler.

Firstly, the level of nesting is controlled by the indentations.
Reason – code with different indents is hard to read and understand,
and code with correct indents is clear without parentheses. The
size of indent is strict – two spaces. It was done for big decoration 
hierarchy. Also, EOlang strictly controls all space symbols – no more
than one whitespace between words, no more than one blank line between 
objects and attributes, no blank lines in the code of one attribute,
no tabs. If a code looks ugly without extra blank lines, it is a good 
marker that you should create a new object or attribute.

EOlang controls not only indentations, but also naming. You can’t 
create an object or a attribute with first capital letter.

EO compiler checks all useless includes, and forces the programmer to remove them.
