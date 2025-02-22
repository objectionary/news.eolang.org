---
layout: post
date: 2025-02-21
title: "Auto named abstract objects or how to reach the ρ"
author: maxonfjvipon
---

It's been a difficult year... It's been a while since our 
[last](https://news.eolang.org/2024-05-14-rho-sigma-delta-lambda.html) blog post. Today, we're back 
and starting by answering a question from our Telegram [chat](https://t.me/eolang_org) reader: 
"What does the `>>` EO syntax stand for"?

<!-- more -->

In one of the previous releases, we introduced this EO syntax for the automated naming of abstract 
objects. The only place where such auto-named abstract objects are allowed is as arguments of an 
application. They CAN'T be used as top-level abstract objects because that wouldn't make any 
sense — there would be no way to "touch" them from other objects.

```
[x] >>     # prohibited

malloc.of
  12
  [m] >>   # allowed
```

The idea is simple: generate a random unique name for an object if the user considers 
meaningful naming unnecessary. But why might they decide this? That's the most interesting part.

To understand the real reason, we need to go back to the 
[previous](https://news.eolang.org/2024-05-14-rho-sigma-delta-lambda.html) blog post, 
particularly the section about the `ρ` (Rho) attribute. Let's recap some important points:

1. The `ρ` (Rho) attribute of the object `voice` refers to the object `animal` that uses `voice`.
2. In EO, the `ρ` (Rho) attribute is indicated by the `^` sign.
3. "The object `animal` uses the object `voice`" implies dynamic dispatch.
4. Dynamic dispatch in EO is a mechanism for retrieving an attribute from an object. 
   Syntactically, it is implemented via dot notation: `animal.voice`.
5. The `ρ` (Rho) attribute is initially empty and is set right after dynamic dispatch occurs.

```
[name] > animal
  [] > voice

animal "kitty" > cat
cat.voice > meow
```

In the code snippet above, we copy the object `animal` and set its attribute `name` to `"kitty"` 
(which is a `string` object). Then, we take the `voice` attribute from the object `cat`. 
At the exact moment the object `voice` is retrieved, its `ρ` (Rho) attribute is set and starts 
referring to the object `cat`. Now, we can do `meow.^`.

Considering the above, we can assume that for the object `meow` (which is a copied `voice` object) 
to use the `ρ` (Rho) attribute, `voice` must be "taken" (or dispatched) from somewhere 
(specifically, from the object `cat`). If it has to be "taken," that means the object `cat` must 
have an attribute named `voice`, because dynamic dispatch is only possible via an attribute name. 
This is the key point: an object must have an attribute with a name. Only the presence of a name 
allows the object to have the `ρ` (Rho) attribute.

Now, let's return to the example with abstract objects.

```
[] > foo
  "Hello" > greetings
  malloc.of > @
    5
    [m]
      $.^.greetings > str
      m.write str > @
```

Here, we have an object `foo` with two attributes: `greetings` and `@`. The attribute `@` is bound 
to the object `malloc.of`, which is copied with two arguments: the object `5` and a nameless 
abstract object (let's call it "scope"). The "scope" has three attributes: `m`, `str`, and `@`. 
As you can see, `str` is bound to a chain of dynamic dispatches: `$.^.greetings`.

Here, `$` means "this" (i.e., the abstract object itself, "scope"); 
`.^` takes the `ρ` (Rho) attribute of the abstract object; 
and `.greetings` retrieves the `greetings` attribute.

Clearly, "scope" is trying to access the `greetings` attribute of `foo` via its `ρ` (Rho) attribute.
For this to work, "scope" must have the `ρ` attribute referring to `foo`, meaning that "scope" must 
be dispatched (or "taken") from `foo`. However, "scope" lacks a name. No name means no dynamic 
dispatch, and without dynamic dispatch, there is no `ρ` (Rho) attribute.

The solution is simple: give a name to "scope".

```
[] > foo
  "Hello" > greetings
  malloc.of > @
    5
    [m] > scope
      $.^.greetings > str
      m.write str > @
```

This is a sugared version where the name `scope` is placed at a certain nesting level. 
To better understand it, let's rewrite it in its canonical form, which is semantically the 
same (and which our compiler actually builds under the hood).

```
[] > foo
  "Hello" > greetings
  [m] > scope
    $.^.greetings > str
    m.write str > @
  malloc.of > @
    5
    $.scope
```

Now, you can see that `foo` has one more attribute, `scope`, and the second argument of 
`malloc.of` now appears slightly different: `$.scope`. Take a closer look at it. 
What do you see? Right! This is dynamic dispatch we have here! The `$` means "this" 
(i.e., the current abstract object, `foo`), and `.scope` retrieves the `scope` attribute 
from `foo`. This means that right after the dispatch is done, the `scope` object has 
its `ρ` attribute set, referring to `foo`, and `$.^.greetings` starts working correctly.

This solves the main problem. However, the canonical form is somewhat verbose, while the 
sugared form relies on attribute naming when it's unnecessary.

For cases where an abstract object’s name is unimportant but must be present to enable dynamic
dispatch, we introduced the `>>` syntax.

```
[] > foo
  "Hello" > greetings
  malloc.of > @
    5
    [m] >>
      $.^.greetings > str
      m.write str > @
```

Under the hood, the compiler rewrites it into something like this:

```
[] > foo
  "Hello" > greetings
  [m] > random-unique-name
    $.^.greetings > str
    m.write str > @
  malloc.of > @
    5
    $.random-unique-name
```

It continues to work according to the same rules but looks cleaner for the user.

That's all for today. Hopefully, you now have a better understanding of the `ρ` attribute and 
dynamic dispatch in EO.

