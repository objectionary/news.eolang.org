---
layout: post
date: 2024-05-14
title: "Rho, Sigma and Other Fantastic Beasts of EO"
author: maxonfjvipon

---

Since the last blog post, we [released](https://github.com/objectionary/eo/releases/tag/0.38.0) a
new version of EO where we got rid of the `σ` (Sigma) attribute. So, this blog post will try to
explain all special attributes and assets such as `Δ` (Delta), `φ` (Phi), `σ` (Sigma), `λ` (Lambda),
and `ρ` (Rho) as promised in one of the previous blog posts.

<!--more-->

### Where I Was Born

The first special attribute we're observing (and which was removed) is `σ` (Sigma). The `σ`
attribute of the object `X` was the attribute that referred to the object `Y` inside which the
scope object `X` was born (formed or created for the first time).

The `σ` attribute in EO:
- was indicated by the `&` sign.
- was initialized right after the object is formed.
- every object except the global parent object `Φ` had it.

For example, `int.& -> eolang`, `float.div.& -> float`. Consider the next code snippet:

```
[] > loop
  "Hello, world" > str

  while > @
    true
    [i]
      stdout > @
        &.str
```

Here we have an endless while loop. The second argument of the object `while` is an anonymous
abstract object, let's call him `X`. The `X` object is not used in the scope of the object `loop`;
it's just created here. The object `X` will be used inside the scope of the object `while` when
dataization is started. But as you may see `X` has access to the scope of the object `loop` via
`&` and may reach the attributes of `loop` like `str`.

And we decided that it was a bad idea to give the object such an opportunity to have access to the
place where it was born. It kind of breaks the idea of object orientation because `&` is actually
a static attribute like a static method in Java. We don't tolerate static methods and attributes,
[here's](https://www.yegor256.com/2014/05/05/oop-alternative-to-utility-classes.html) why.

### Who Uses Me

The second special attribute we're observing is `ρ` (Rho). The `ρ` attribute of the object `X` is
the attribute that refers to the object `Y` that uses the object `X`. In EO, the attribute `ρ` is
indicated by the `^` sign. Drawing the analogy with Java, the closest thing to `ρ` is the `this`
keyword which refers to the current object:

```
class Animal {
  String type;
  Animal(String tpe) {
    this.type = tpe;
  }
  void voice() {
    System.out.print("I'm a %s", this.type)
  }
}

Animal cat = new Animal("Cat");
cat.voice();                    // I'm a Cat

Animal dog = new Animal("Dog");
dog.voice();                    // I'm a Dog
```

Here when we create two instances of `Animal` - `cat` and `dog`. The `this` keyword inside their
functions `voice` refers to different objects - `cat` and `dog` accordingly.

The same functionality can be achieved in EO:
```
[type] > animal
  [] > voice
    stdout > @
      sprintf
        "I'm a %s"
        ^.type

animal "Cat" > cat
cat.voice > moew   # I'm a Cat

animal "Dog" > dog
dog.voice > woof   # I'm a Dog

```
The main difference between `this` in Java and `ρ` in EO is the moment when these "links"   actually
become referred to the objects. In Java - right after the object is created, in EO - on attribute
dispatch. Let's look a bit closer. The dynamic dispatch in EO is a mechanism of retrieving an
attribute from the object. Syntactically it's implemented via "dot-notation".

So this is dispatch:

```
cat.voice
```

We're trying to retrieve an attribute `voice` from the concrete object `cat`. At the moment we've
found the attribute `voice` inside the object `cat` and ready to return it - the `voice` attribute
is copied and its `ρ` attribute is initialized with a link to the object `cat`. Until we touch
`cat.voice` object its `ρ` attribute refers to `Ø` (nothing).

A few more examples:

```
cat.voice > voice1  # voice1.^ -> cat
cat.voice > voice2  # voice2.^ -> cat
dog.voice > voice3  # voice3.^ -> dog

cat.voice > voice4  # voice4.& -> animal - deprecated
dog.voice > voice5  # voice5.& -> animal - deprecated
```

### What I Decorate

The third special attribute we're observing is `φ` (Phi). We've already described the attribute in
one of the previous blog posts, but let's dive a bit deeper. In EO, the attribute is indicated
by `@` sign. The attribute is not mandatory and may be absent. The `φ` attribute of the object `X`
is the attribute that refers to the object `Y` which object `X` decorates. The main purpose of
decoration - reuse of the attributes. For example:

```
[type] > animal
  [] > voice
    stdout > @
      sprintf
        "I'm a %s"
        ^.type

[] > cat
  animal "Cat" > @
```

Here we have an object `cat` which decorates an object `animal` with `type` attribute set to
`"Cat"`. That means that all the attributes which are allowed to be taken from the object `animal`,
like `voice`, are allowed to be taken from the object `cat`:

```
cat.voice > meow # I'm a Cat
```

The decoration may have several layers and all the attributes on the deepest level are available
on the top level:

```
[type] > animal
  [] > voice
    stdout > @
      sprintf
        "I'm a %s"
        ^.type

[color] > cat
  animal > @
    sprintf
      "%s cat"
      color

[] > black-cat
  cat "black" > @
```

Here the object `black-cat` decorates the object `cat` and the object `cat` decorates the object
`animal`. This onion of decorators allows taking the attribute `voice` from the object `black-cat`:

```
black-cat.voice > meow # I'm a black cat
```

### What Data I Have

The fourth special thing we're observing is `Δ` (Delta) asset. A few words about this asset:
- It's not an attribute but an asset because it refers not to the object but to the data which is
  a sequence of bytes.
- There's no way to explicitly touch this asset in EO.
- Only `org.eolang.bytes` object has this asset.
- The only way to touch the data in the asset is dataization.

### What I Can Reach from Outside

The last special thing we're observing is `λ` (Lambda) asset. Let's look at it a bit closely:
- It's not an attribute but an asset because it refers not to the object but to some external
  function which returns an object.
- Objects in EO that have the `λ` asset are called "atoms".
- There's no way to explicitly touch this asset in EO.
- Atoms can't have a `φ` attribute.
- The `λ` asset, as well as the `φ` attribute, also allows reusing the attributes:

```
[] > mars-termerature /float # measures the temperature on Mars and returns float

mars-temperature.div 10 > divided
```

Here, as you may see, `mars-termerature` is the atom that does not have an attribute `div` and
returns `float`. However, we can still retrieve the attribute `div` from it. It will go to the
`λ` asset, execute it, do some calculations, return us some `float`, and the `div` attribute will
be taken from this `float`.

That's all for today. We'll be right back in a week with a new fresh blog post. Stay in touch.
