---
layout: post
date: 2024-05-02
title: "New Syntax for Nameless @-Bound-Only Objects"
author: maxonfjvipon

---
We're continuing to observe the features of the latest 
[release](https://github.com/objectionary/eo/releases/tag/0.37.0) of EO, and today we're talking 
about new syntax for nameless @-bound-only objects.

<!--more-->

The `@` attribute is a special attribute that indicates the current object decorates another object.
For example, in the snippet below, objects `cat` and `dog` decorate an object `animal`:

```
[voice] > animal

[] > cat
  animal "Meow" > @

[] > dog
  animal "Woof" > @
```

The attribute `@` is also often used in nameless abstract objects. A common example is the second 
argument of the object `try`:

```
try
  1.div 0
  [e]
    QQ.io.stdout > @
      e
  true
```

We noticed quite some time ago that there are so many small objects with only the `@` attribute 
bound. So, to simplify the code and reduce the number of indentations, we introduced new syntax for 
such objects:

```
QQ.io.stdout e > [e]

# Which is the same as

[e]
  QQ.io.stdout e > @
  
# OR

[e]
  QQ.io.stdout > @
    e
```

It's worth noting that only objects in horizontal notation can be used with such syntax:

```
QQ.io.stdout e > [e]       # horizontal application, the same as:
                           # [e]
                           #   QQ.io.stdout e > @
                           #
if. true first second > [] # reversed horizontal application, the same as:
                           # []
                           #   if. > @
                           #     true
                           #     first
                           #     second
                           # OR
                           # []
                           #   if. true first second > @
                           #
QQ.io.stdout > []          # horizontal method/dispatch, the same as:
                           # []
                           #   QQ.io.stdout > @
                           #
x > []                     # object reference, the same as:
                           # []
                           #   x > @
                           # OR
                           # [] (x > @)
                           #
[x] (5.plus x > sum) > [i] # horizontal anonymous abstract object
                           # the same as:
                           # [i]
                           #   [x] > @
                           #     5.plus x > sum
```

All objects in vertical notation can NOT be used with such syntax and their usage leads to a parsing 
exception:
```
QQ.io.stdout > [e]         # vertical application - wrong
  e                        #
                           #
QQ                         # vertical method/dispatch - wrong
.io                        #
.stdout > [e]              #
                           #
if. > @                    # reversed vertical method with application - wrong
  true                     #
  first                    #
  second                   #
                           #
[x] > [i]                  # vertical anonymous abstract object - wrong
  5.plus x > sum           #
```

Returning to the example with object `try`, it can now be written in a shorter and more convenient way:
```
try
  1.div 0
  QQ.io.stdout e > [e]
  true
```

That's all for today. We'll be right back in a week with a new fresh blog post. Stay in touch.