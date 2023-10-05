---
layout: post
date: 2023-10-04
title: "Application to application"
author: maxonfjvipon
---

In the process of working on the recent release 
[0.32.0](https://github.com/objectionary/eo/releases/tag/0.32.0), we've faced an interesting case 
related to one of the fundamental concepts of EO in particular and phi-calculus in general - 
application.

This blog post will attempt to explain what application is under the hood, how it works in our 
compiler, and why this code `(arr.at 0) "Hello"` does not work in the way you expect.

<!--more-->

### Application
Application is the process of copying an abstract object while specifying some of its free 
attributes.

Consider the example. Here, `dog` is an abstract object with one free attribute: `name`.

```
[name] > dog
  stdout > bark
    sprintf
      "I'm %s! Woof!"
      name
```

Abstract objects are like templates or factories for concrete objects. So, getting a specific 
instance of a `dog` occurs in two stages: copying and setting free attribute. The copying process 
takes place behind the scenes. Thus, we obtain a new specified object, `dog`, with its `name` as 
"Bary".

```
dog "Bary" > bary
```

At the Java level (into which EO is being translated), application looks something like this:

```java
Phi dog = Phi.Ф.attr("org.eolang.dog").get(); // finding an abstract object "dog"
Phi copy = dog.copy();                        // copying/cloning an abstract object
copy.attr("name").put("Bary");                // setting the free attribute
```

As you can see, no object execution occurs. It's vital to note and remember that everything that 
happens during the application involves getting a new object by copying and setting its internal 
state. Nothing more.

### Application to application

However, despite knowing what application is, we encountered an interesting problem for which we 
couldn't find a solution for a long time.

Consider the following code:

```
1. [name] > dog
2.   "I'm %s! Woof!" > bark
3. * dog > arr
4. (arr.at 0) "Bary" > bary
```

- As before, `dog` is an abstract object with one free attribute, `name`.
- In the third line, an abstract object `dog` is stored in an array.
- In the fourth line, we attempt to retrieve an abstract object `dog` from the array by applying 
object `0` to `arr.at` (in simpler words, we're trying to access the first element of the array) 
and then set the object `"Bary"` as the free attribute `name` of the `dog`.

We can try to clarify the code like this:

```
1. [name] > dog
2.   "I'm %s! Woof!" > bark
3. * dog > arr
4. arr.at 0 > first
5. first "Bary" > bary
```

It appears that everything is fine, and the code should work, but it doesn't, and moreover, it must 
not.

### What's wrong

It all comes down to the application discussed earlier. No object execution occurs during the 
application. So if you take a closer look at the fourth line:

```
4. arr.at 0 > first
```

there is no information about what's inside the array. It simply sets the object `0` as the free 
attribute of the object `arr.at`, and that's it.

And what happens next in the fifth line:

```
5. first "Bary" > bary
```

We're taking an object `first` (which is just a reference to the object `arr.at` with an already 
specified free attribute) and then trying to specify its free attribute again.

And obviously (perhaps not so obvious), this leads to an error because we're trying to set the same 
free attribute twice.

### So, what's the solution?

Firstly, code where application is placed at the "head" of another application is now prohibited at 
the grammar level:

```
(arr.at 0) "Bary"
```

Secondly, we found out that such cases can be easily resolved with code like this:

```
(arr.at 0).@ "Bary" > bary
```

Here, we first specify the free attribute of the object `arr.at`, and then we access its `@` 
attribute and apply the object `"Bary"` to it.

The information about the abstract object `dog` stored inside the array is revealed when we access 
the `@` attribute of the object `arr.at`.

In this particular case, `arr.at.@` is a so-called lambda object that performs calculations and 
returns the object from the array by providing an index. It's worth noting that if the free 
attribute of `arr.at` is not specified, the following code:

```
arr.at.@
```

will fail with an error.

I hope you've learned a little more about EO today. Stay tuned for updates and be in touch!