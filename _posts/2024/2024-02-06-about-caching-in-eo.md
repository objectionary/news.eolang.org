---
layout: post
date: 2024-02-06
title: "Build cache in EO and other build systems"
author: Alekseeva Yana
---


## Introduction 
In [EO](https://github.com/objectionary/eo), caching is used to speed up program compilation.
Recently we found a caching 
[bug](https://github.com/objectionary/eo/issues/2790) between goals in `eo-maven-plugin`
for EO version `0.34.0`. The bug occurred because the old verification method
used compilation time and caching time to search for a cached file.
This is not the most reliable verification method,
because caching time does not have to be equal to compilation time.
We came to conclusion that we need caching with a reliable verification method.
Furthermore, this verification method should refrain from reading the file content.

The goal is to implement effective caching in EO.
To achieve the goal, we will briefly look at how well-known used build systems (such as ccache, Maven, Gradle)
in order to gain a deeper understanding of the caching concepts employed within them.

<!--more-->

## Caching in Other Build Systems

### ccache/sccache
In compiled programming languages, building a project with many source code files takes a long time.
This time is spent on loading of libraries, preparing, optimizing, checking the code, and so on.
Let's look at the assembly scheme using C++ as an example [Picture 1](/images/defaultCPhase.svg):

<p align="center">
  <img src="/images/defaultCPhase.svg">
</p>

1) First, preprocessor retrieves the source code files,
which consist of both source files `.cpp` and header files `.h`.
The result is a single file `.cpp` with human-readable code that the compiler will get.
2) The compiler receives the file `.cpp` from the preprocessor and compiles it into an object file - `.obj`.
At the compilation stage, parser checks whether the code matches rules of a specific programming language. 
To speed up compilation, different files of the same project might be compiled in parallel.
3)  Then, the [Linker](https://en.wikipedia.org/wiki/Linker_(computing)) combines object files
into an executable `.exe` file.


To speed up the build of compiled languages, [ccache](https://ccache.dev)
and [sccache](https://github.com/mozilla/sccache) are used.    
`ccache` uses the hash algorithm for the hashing of code at certain stages of the build.
When compiling a file, its hash is calculated. 
If the file is already present in the registry of compiled files, the file will not be compiled again.
Instead, the previously compiled binary file will be utilized.
This approach can significantly accelerate the build process of certain packages.
The [`ccache` hash](https://ccache.dev/manual/4.8.2.html#_common_hashed_information) is
based on:
* the file contents
* the current directory of the file
* the name of the compiler
* the compiler’s size and modification time
* extensions used by the compiler.

Moreover, `ccache` has two types of the hashing:
1) `Direct mode` - the hash is generated based on the source code only.
When using this mode, the user must ensure that the external libraries used in a project have not changed.
Otherwise, the project might fail to build, resulting in errors.
2) `Preprocessor mode` - hash is generated based on the `.cpp` file received after the preprocessor step.


`Sccache` is similar in purpose to `ccache` but provides more functionality.
`Sccache` allows to store cached files not only locally, but also in a cloud data storage.
And `sccache` supports a wider range of languages, while `ccache` focuses on caching C and C++ compiler.


`ccache` cannot work with individual compilation tasks (e.g. `Maven goal` or `Gradle task`).
However, the hashing approach and the concept of non-local data storage could potentially
be incorporated during the development of the EO caching mechanism.


### Gradle
[Gradle](https://gradle.org) builds projects using a
[task graph](https://docs.gradle.org/current/userguide/build_lifecycle.html) that allows for synchronous execution 
of certain tasks. A task represents a unit of work in `Gradle` project.


`Gradle` employs
[Incremental build](https://docs.gradle.org/current/userguide/incremental_build.html#sec:how_does_it_work),
to speed up project builds.
To enable an incremental build, the project tasks must specify their input and output files.
`Incremental build` uses a hash to detect changes in the inputs and the outputs.
The single hash contains the paths and the contents of all the input files or output files.
1) Before executing a task, `Gradle` takes a hash of the input files and saves it.
   The hash is considered valid if the last modification time and the size of the source files have not changed.
2) Then `Gradle` executes the task and saves a hash of the output files.
3) Then, when Gradle starts a project build again, it generates a new hash for the same files.
   If the new hash is valid, Gradle can safely skip this task. 
   In the opposite case, the task performs an action again and rewrites outputs.


In addition to `Incremental build`, `Gradle` also stores hash of each previous build, enabling quick project builds,
for example when switching from one git branch to another. This feature is known as 
the [Build Cache](https://docs.gradle.org/current/userguide/build_cache.html).


`Gradle Incremental build` can manage separate compilation tasks based on inputs and outputs.
And the EO compiler consists of a unit of work in `Maven` (the last section contains a detailed description).
Steps of the EO compiler can have input and output files.
Building upon the concept of `Gradle Incremental Build`, we can use its principles to develop the EO caching mechanism.


### Maven
[Maven](https://maven.apache.org) automates and manages Java-project builds.
`Maven` is based on the concept of 
[Maven LifeCycles](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html), 
which includes default, clean, and site lifecycles.

In Maven, there are default phases and goals for building any projects:

<p align="center">
  <img src="/images/defaultPhaseMaven.svg">
</p>

In Maven, the `phases` are inherently interconnected within the build lifecycle.
A `phase` represents a specific task, and the execution order of `phases` is determined by the default Maven 
lifecycle bindings. Each `phase` functions as a series of individual tasks known as `goals`.
There are `goals` tied to the Maven lifecycle, as shown in [Picture 2](/images/defaultPhaseMaven.svg).
It's also possible to add a new `goal` to a desired phase by modifying the `pom.xml` file. 
Additionally, Maven also supports `goals` that are not bound to any build phase
and can be executed outside the build lifecycle, directly through the command line.


`Maven` can utilize caching mechanisms through the `takari-lifecycle-plugin` and `maven-build-cache-extension`:

* The [takari-lifecycle-plugin](http://takari.io/book/40-lifecycle.html) is an alternative to the default Maven lifecycle 
(building JAR files). Its distinct feature lies in the use of a single universal plugin with the equivalent
functionality to plugins for the standard lifecycle, but with significantly fewer dependencies. This plugin leverages 
[The Takari Incremental API](https://github.com/takari/io.takari.incrementalbuild), 
which introduces the concept of `builders`. These `builders` are user-provided public non-abstract
top-level classes that implement specific build actions.
They can produce various types of outputs, including generated/output files on the filesystem, 
build messages, and project model mutations. For each `builder` annotated method, a maven mojo, 
which represents a maven `goal`, is generated.
When a `builder` is run for a given set of inputs, it produces and saves to the specified directory the same outputs. 
Any changes in the inputs result in the removal of outputs.


* The [maven-build-cache-extension](https://maven.apache.org/extensions/maven-build-cache-extension/)
is utilized for large Maven projects that have a significant number of small `modules`.
A `module` refers to a subproject within a larger project.
Each `module` has its own `pom.xm` file, and there is an aggregator `pom.xml` that consolidates all the `modules`.
This plugin takes a hash from `module` inputs and stores outputs in the cache.
The cache restores unchanged `modules`.  
In the opposite case, the cache seamlessly delegates the build work to the standard Maven core,
without interfering with the build execution logic.

  
Let's clarify upfront that the Maven Build Cache Extension is not suitable for caching EO compilation stages,
as it is designed for caching at the module level within a project and not for individual tasks.


Special attention should be given to the Takari Incremental API. 
This API can be applied to cache EO compilation stages as it operates with `goals`.
It does not use hashing algorithms, which can slow down project build times,
and it does not have separate cache directories.
Each `builder` has own directories for input and output data related to their work.

### EO build cache

The EO code uses the `Maven` for building projects.
For this purpose, there is the `eo-maven-plugin` containing the essential goals for working with EO code.
As previously mentioned, the build of projects in Maven follows a specific order of phases. 
Below is a diagram illustrating the main phases and their corresponding goals for the EO:

<p align="center">
  <img src="/images/EO.svg">
</p>

In [Picture 3](/images/EO.svg) the goals of the `eo-maven-plugin` are highlighted in green.


However, the actual work with EO code takes place in `AssembleMojo`.
`AssembleMojo` is the goal consisting of other goals that work with the EO file, as shown in
[Picture 4](/images/AssembleMojo.svg).


<p align="center">
  <img src="/images/AssembleMojo.svg">
</p>

Each goal within `AssembleMojo` is a distinct compilation step for EO code.
These goals happen one after the other. Each goal has directories for input and output data,
as well as a directory for storing cached data.
Using the program name, each goal can receive and store data.


The previous caching mechanism in EO made use of distinct interfaces, specifically `Footprint` and `Optimization`.
These caching interfaces shared similar logic, but with minor differences.
For instance, `Footprint` verifies the EO version of the compiler, whereas the remaining checks are identical.
Additionally, the conditions for searching data in the cache had errors.
Due to this issue, the program behaved incorrectly, because saving the goal's result to the cache is not instantaneous.
After conducting an in-depth analysis of the project's incorrect operation,
several disadvantages of the previous EO caching mechanism were brought to light:
* Incorrect search conditions for data in the cache.
* The verification method requires reading the file content, which results in inefficiencies.
* The presence of multiple caching mechanisms creates challenges in identifying and rectifying caching errors.
* Employing multiple caching mechanisms for similar entities is a suboptimal practice, 
leading to redundancy and complicating the caching infrastructure.


In tackling caching challenges within EO, we conducted a thorough evaluation of current caching systems.
Most existing caching systems are not suitable for the EO project.
However, one candidate emerged as a potential solution for caching EO compilation stages: the Takari Incremental API.
The Takari Incremental API exhibits key similarities with the EO caching system,
notably in its utilization of inputs and outputs directories, absence of a hash for data storage and retrieval,
and compatibility with Maven goals.
However, it diverges from the EO caching approach in one significant aspect – the absence of a distinct cache directory.

We can try to use this API or implement our own caching approach, correcting the disadvantages found.
The envisioned approach involves the creation of a singular class responsible
for storing and retrieving data from the cache.
The logic for checking the relevance of cached data is presented below:
1) We create EO program, named "example".
   Intermediate files during compilation of this program will have the same name, but not the format
   (e.g. `example.eo`, `example.xml`).
2) When the EO compiler compiles this program task, it saves files of compilation steps into cache.
   Each compilation step has its own caching directory.
3) When the EO compiler starts a project build again, it will check if there is a file, named "example",
   in the cache of step. If such a file exists,
   then it is enough to check that the last modification time of this file at the current step
   is later than at the previous step. If this condition is true,
   then the finished file can be retrieved from the cache.
   Below is a diagram illustrating the EO compilation steps, which have caching directory for EO version `0.34.0`:

<p align="center">
  <img src="/images/SavingInCacheEO.svg">
</p>

4) If the EO program file [Picture 5](/images/RewritingInCacheEO1.svg) 
   or an intermediate file [Picture 6](/images/RewritingInCacheEO2.svg) have changed,
   then the previously cached files become invalid.
   In this case, the compilation step performs an action again and rewrites outputs.

<p align="center">
  <img src="/images/RewritingInCacheEO1.svg">
</p>


<p align="center">
  <img src="/images/RewritingInCacheEO2.svg">
</p>












