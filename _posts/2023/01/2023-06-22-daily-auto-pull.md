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
Until recently, the last stage was performed manually with the help of a script that added modified files by the 
library's URL, after which a pull request was created. And that stage consisted of the following steps:

1.To create a pull request in `objectionary/home` by `pull.sh` script in a separate git branch (the name of the branch doesn't matter):
   ```shell
   ./pull.sh objectionary/eo
   ```
This update all `.eo` files in `gh-pages` from `objectionary/eo`. 
It also changes the corresponding versions of the eo objects (e.g. from the `+version` metadata) in the `objectionary/home` repository.

2.To check and to remove manually either unused or old files. Script didn't do it.

3.To update `eo.version` in `pom.xml` in `objectionary/home`.

4.Finally, all `todo` must be removed manually.

To avoid manual changes we created auto pulling daily run script. It runs once an hour every day to check for new releases of [Objectionary](https://github.com/objectionary/) libraries. 
If a new release is found, the script creates a new pull request with corresponding changes. This process ensures that our codebase 
is always up-to-date and optimized.

The benefits of this automated system are numerous. First, it saves developers time by eliminating the 
need to manually check for new releases. Second, it ensures that our codebase is always up-to-date with 
the latest features and optimizations. Finally, it helps us maintain a high level of code quality by 
ensuring that we are using the most recent versions of our libraries.
