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

Our daily run script runs at `00:00` every day to check for new releases of [Objectionary](https://github.com/objectionary/) libraries. 
If a new release is found, the script creates a new pull request with corresponding changes. This process ensures that our codebase 
is always up-to-date and optimized.

The benefits of this automated system are numerous. First, it saves developers time by eliminating the 
need to manually check for new releases. Second, it ensures that our codebase is always up-to-date with 
the latest features and optimizations. Finally, it helps us maintain a high level of code quality by 
ensuring that we are using the most recent versions of our libraries.
