---
layout: post
date: 2023-06-22
title: "Automating the Pull of New Releases for EO Libraries"
author: graur
---

As a developer, keeping up with the latest releases of libraries is crucial to ensure that your code is
up-to-date and optimized. However, manually checking for new releases can be time-consuming and tedious.
That's why we've created an automated pull system for new releases of EO libraries in our 
[Home](https://github.com/objectionary/home) repository.

<!--more-->

The process of creating a release of EO libraries consists of three main stages: releasing the EO library, 
publishing it on Maven Central and on the GitHub of this library in the `gh-pages` branch, and adding EO objects from 
the `gh-pages` branch of this library to [Objectionary Home](https://github.com/objectionary/home) repository. 
Until recently, the last stage was performed manually with the help of a script that added modified files by the library's
URL, after which a pull request was created.

To avoid manual changes we created auto pulling daily run script. It runs at `00:00` every day to check for new releases of [Objectionary](https://github.com/objectionary/) libraries. 
If a new release is found, the script creates a new pull request with corresponding changes. This process ensures that our codebase 
is always up-to-date and optimized.

The benefits of this automated system are numerous. First, it saves developers time by eliminating the 
need to manually check for new releases. Second, it ensures that our codebase is always up-to-date with 
the latest features and optimizations. Finally, it helps us maintain a high level of code quality by 
ensuring that we are using the most recent versions of our libraries.
