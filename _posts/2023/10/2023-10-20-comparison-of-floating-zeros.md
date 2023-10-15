---
layout: post
date: 2023-10-20
title: "Comparison of 0.0 and -0.0"
author: c71n93
---

Due to the peculiarities of working with data in EO, an interesting quirk arises when comparing `0.0` and `-0.0`. The 
fact is that in EO these two values are not equal.

This short blog post provides a simplified explanation of number encodings and how the comparison of `0.0` and `-0.0` 
takes place in EO and other popular programming languages.

### Why do we have two zeros?

Without delving into the intricacies of number encoding and standards, let's briefly understand why this happens.

The familiar `signed int` works based on the [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) 
principle, in which zero has a unique representation (all bits are `0`).

On the other hand, `float` operates differently, using the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard 
for encoding and arithmetic operations. In this standard, numbers have a sign bit. If the sign bit is `1`, the number is 
negative; if it's `0`, the number is positive. As a result, we have two different binary representations of float 
zero: `0.0` has all zeros in its binary representation, while `-0.0` has all zeros except for the sign bit.

### Comparison of 0.0 and -0.0 in EO

Currently, in EO comparison behaves as follows:

```
0.0.eq -0.0 # FALSE
0.0.gt -0.0 # FALSE
0.0.lt -0.0 # FALSE
0.0.gte -0.0 # FALSE
0.0.lte -0.0 # FALSE
```

This happens because the `eq` attribute compares data within objects bitwise. In other words, regardless of the data 
being compared, it is first interpreted as a set of bits and then compared.

Thus, due to the fact that numbers `0.0` and `-0.0`, as previously determined, have different bitwise representations, 
when compared in EO using `eq`, they turn out to be not equal.

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

So why does this happen if `0.0` and `-0.0` have different bitwise representations? The same IEEE 754 standard states 
that *"negative zero and positive zero should compare as equal with the usual (numerical) comparison operators"* 
(referencing the standard in the blog post isn't possible as it's a paid document, but you can find this information 
[here](https://en.wikipedia.org/wiki/Signed_zero)).

Hence, comparison operators such as `==` in the languages discussed above should modify their semantics for comparing 
`float` values to adhere to the standard. This is indeed the case in most languages.

### Conclusion

- *What strategy will we follow and why?*
- *Are there any advantages to the EO approach?*
