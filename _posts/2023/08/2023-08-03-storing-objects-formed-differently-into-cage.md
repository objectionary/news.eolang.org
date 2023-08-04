---
layout: post
date: 2023-08-03
title: "The CAGE Prohibits Storing Objects Formed Differently"
author: maxonfjvipon
---

In the recently released version [0.30](https://github.com/objectionary/eo/releases/tag/0.30)
we've changed the writing mechanism of `cage` object. Until now, we could store and write to 
`cage` any object we wanted. But `cage` became smarter and stricter and can store only objects 
that have the same "form" now.

<!--more-->

A few words about "form" and "formation." Let's take a look at the code below:

```
[name] > cat
  [] > voice
    stdout
      sprintf
        "Meow from %s"
        name
```

This is a "formation" of the object `cat`. Here we can say that the "form" of object `cat` is "cat."

```
cat "Lisa" > lisa
```

Here object `cat` was copied and its free attribute `name` was set to "Lisa." Now we can say that
object `lisa` "was formed by" object `cat` and has the same "form"—"cat."

Let's get back to the object `cage`. The object is used as temporary storage of objects in memory.
It should be remembered that `cage` does not dataize a stored object.

Here is the typical example how `cage` was used before:

```
cage 0 > cg

seq > @
  cg.write lisa
```

Here we copy object `cage` and store the object `0` (which is "formed by" object `int`) in it, and 
then we write an object `lisa` to it.

Such behavior is prohibited now and will lead to exception throwing.
Object `cage` can store inside only objects of the same "form." 

The next code will work since object `cat` and `lisa` have the same "form"–"cat":

```
cage cat > cg

seq > @
  cg.write lisa
```

You may wonder why we did it.
Here we talk a bit about our plans for EO.  

We think that EO can be used as intermediate representation for performing optimizations for many 
object-oriented programming languages.
To achieve that, we have to make EO stronger and stricter.
This is how we're about to do it:
- [x] Prohibit writing objects formed differently into `cage`
- [ ] Prohibit writing objects formed differently into `memeory`
- [ ] Prohibit weak atoms typing (`[] > atom /?`)
- [ ] Maybe anything else...

Be in touch and follow the news
