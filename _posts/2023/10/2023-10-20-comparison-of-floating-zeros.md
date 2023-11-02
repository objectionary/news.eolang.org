---
layout: post
date: 2023-11-02
title: "Comparison of 0.0 and -0.0"
author: c71n93
---

Due to the peculiarities of working with data in EO, an interesting quirk had been arising when comparing `0.0` and 
`-0.0.` The fact is that in EO, these two values were not considered equal until we made changes.

Until recently, the comparison of `0.0` and `-0.0` in EO didn't work like in other languages, but we changed that. This 
short blog post provides a simplified explanation of number encodings, how such comparison takes place in popular 
programming languages, and how we changed this comparison in EO to meet the standard.

<!--more-->

### Why do we have two zeros?

Without delving into the intricacies of number encoding and standards, let's briefly understand why this happens.

The familiar `signed int` works based on the [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) 
principle, in which zero has a unique representation (all bits are `0`).

On the other hand, `float` operates differently, using the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard 
for encoding and arithmetic operations. In this standard, numbers have a sign bit. If the sign bit is `1`, the number is 
negative; if it's `0`, the number is positive. As a result, we have two different binary representations of float 
zero: `0.0` has all zeros in its binary representation, while `-0.0` has all zeros except for the sign bit.

### Comparison of 0.0 and -0.0 in EO before the changes

EO comparison used to work as follows:

```
0.0.eq -0.0 # FALSE
0.0.gt -0.0 # FALSE
0.0.lt -0.0 # FALSE
0.0.gte -0.0 # FALSE
0.0.lte -0.0 # FALSE
```

This happened because the `eq` attribute compared data within objects bitwise. In simpler terms, regardless of the data 
being compared, it was first interpreted as a set of bits and then compared.

Thus, due to the fact that numbers `0.0` and `-0.0`, as previously determined, have different bitwise representations, 
when compared in EO using `eq`, they turned out to be not equal.

### Comparison of 0.0 and -0.0 in other languages

In most mainstream programming languages `0.0` is considered equal to `-0.0` in basic equality comparisons. For 
example, in Java, C++, JavaScript, and Python, the comparison of these values works as follows:

```java
0.0 == -0.0 // true
0.0 < -0.0 // false
0.0 > -0.0 // false
0.0 <= -0.0 // true
0.0 >= -0.0 // true
```

Why do these languages consider `0.0` and `-0.0` as equal, even though they have different bitwise representations? The 
same IEEE 754 standard states that *"negative zero and positive zero should compare as equal with the usual (numerical) 
comparison operators"* (referencing the standard in the blog post isn't possible as it's a paid document, but you can 
find this information [here](https://en.wikipedia.org/wiki/Signed_zero)).

Thus, this behavior for float numbers is built into most CPUs and works at the assembler level. Comparison operators, 
such as `==`, in the languages discussed above, correspond to how it is implemented in the hardware. This is indeed the 
case in most languages.

### Comparison of 0.0 and -0.0 in EO now

In order for the floating-point comparison in EO to comply with the standard, we have changed the behavior of the `eq` 
attribute for `float`. After these changes the comparison of `0.0` and `-0.0` behaves as follows:

```
0.0.eq -0.0 # TRUE
0.0.gt -0.0 # FALSE
0.0.lt -0.0 # FALSE
0.0.gte -0.0 # TRUE
0.0.lte -0.0 # TRUE
```

Now it works like in all popular programming languages. It's worth noting that we didn't add extra atoms (you can read 
more about atoms in [this](https://news.eolang.org/2022-12-02-java-atoms.html) blog post). The `eq` attribute still 
compares floats bitwise; we simply added a special condition for zeros.

### Conclusion

We changed `eq` attribute of `float` to meet the IEEE 754 standard. Significant reasons are needed to deviate from the 
standard, so we decided that it would be more correct to do as in other programming languages.
