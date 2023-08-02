---
layout: post
date: 2023-08-01
title: "The CAGE Prohibits Storing Objects Formed Differently"
author: maxonfjvipon
---

In the recently released version [0.30](https://github.com/objectionary/eo/releases/tag/0.30)
we've changed the writing mechanism of `cage` object. Until now, we could store and write to 
`cage` any object we wanted. But `cage` became smarter and stricter and can store only objects 
that have the same "form" now.

<!--more-->

A few words about "form" and "formation". Let's take a look at the code below:

```
[name] > cat
  [] > voice
    stdout
      sprintf
        "Meow from %s"
        name
```

This is a "formation" of the object `cat`. Here we can say that the "form" of object `cat` is "cat".

```
cat "Lisa" > lisa
```

Here object `cat` was copied and its free attribute `name` was set to "Lisa". Now we can say that
object `lisa` "was formed by" object `cat` and has the same "form" - "cat".

Let's get back to the object `cage`. The object is used as temporary storage of objects in memory.
It should be remembered that `cage` does not dataize stored object.

Here is the typical example how `cage` was used before:

```
cage 0 > cg

seq > @
  cg.write lisa
```

Here we copy object `cage` and store the object `0` (which is "formed by" object `int`) in it, and 
then we write an object `lisa` to it.

Such behaviour is prohibited now and will lead to trowing an exception. Object `cage` can store 
inside only objects of the same "form". 

The next code will work since object `cat` and `lisa` have the same "form" - "cat":

```
cage cat > cg

seq > @
  cg.write lisa
```

You may wonder why we did it... future plans...
