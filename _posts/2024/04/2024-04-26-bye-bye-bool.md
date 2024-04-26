---
layout: post
date: 2024-04-26
title: "Bye, bye, bool!"
author: maxonfjvipon

---
After the [previous](https://news.eolang.org/2024-04-16-release-0-36-0.html) blog post, one of the 
followers brought an interesting suggestion in our Telegram [chat](https://t.me/eolang_org) (join it btw). 
He proposed getting rid of the object `bool` and making `if` an object not an atom. 
And it was quite interesting. So we made a new [release](https://github.com/objectionary/eo/releases/tag/0.37.0) 
where we followed our follower's proposal.

<!--more-->

The idea is simple: instead of one object `bool`, we introduced two new objects, `true` and `false`, 
which decorate specific `bytes`, `01-` and `00-`, respectively.

```
[] > true
  01- > @

[] > false
  00- > @
```

Such implementation has the next benefits:
- There's no way to create a boolean object with an unexpected amount of bytes.
- We got rid of the atom `if` and defined it right inside `true` and `false`:

```
[] > true
  01- > @
  
  [left right] > if
    left > @

[] > false
  00- > @
  
  [left right] > if
    right > @
```

The decision on which `if` branch we should jump to is transferred to the moment of creation of the 
objects `true` or `false`.

- We got rid of two literals `TRUE` and `FALSE` in EO 
  [grammar](https://github.com/objectionary/eo?tab=readme-ov-file#backus-naur-form).
- The implementation of other `bool` objects like `and`, `or`, `not` became simpler.

So, the result `true` and `false` objects now look like:

```
[] > true
  01- > @
  false > not
  
  [left right] > if
    left > @
  
  [x] > and
    01-.eq x > @
  
  [x] > or
    ^ > @

[] > false
  00- > @
  true > @
  
  [left right] > if
    right > @
  
  [x] > and
    ^ > @
  
  [x] > or
    01-.eq x > @
```

That's it for today. There are more interesting features we introduced in the last release. 
We're planning to write blog posts about all of them because they're worth it. So keep following us!