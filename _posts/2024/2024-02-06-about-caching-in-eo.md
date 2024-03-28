---
layout: post
date: 2024-02-06
title: "Build cache in EO and other build systems"
author: Alekseeva Yana
---


## Introduction 
In [EO](https://github.com/objectionary/eo), caching is used to speed up program compilation.
Recently we found a caching 
[bug](https://github.com/objectionary/eo/issues/2790) in `eo-maven-plugin`
for EO version `0.34.0`. The bug occurred because the old verification method
used compilation time and caching time to search for a cached file.
This is not the most reliable verification method,
because caching time does not have to be equal to compilation time.
We came to the conclusion that we need caching with a reliable verification method.
Furthermore, this verification method should refrain from reading the file content.

The goal is to implement effective caching in EO.
To achieve the goal, we will briefly look at how well-known used build systems (such as ccache, Maven, Gradle)
in order to gain a deeper understanding of the caching concepts employed within them.

<!--more-->

## Build caching of existing build systems

### ccache/sccache
In compiled programming languages, building a project with many source code files takes a long time.
This time is spent on loading of libraries, preparing, optimizing, checking the code, and so on.
Let's look at the assembly scheme using C++ as an example:

<p align="center">
  <img src="/images/defaultCPhase.svg">
</p>

1) First, preprocessor retrieves the source code files,
which consist of both source files `.cpp` and header files `.h`.
The result is a single file `.cpp` with human-readable code that the compiler will get.
2) The compiler receives the file `.cpp` from the preprocessor and compiles it into an object file - `.obj`.
At the compilation stage, parsing checks whether the code matches rules of a specific programming language.
At the end, the compiler optimizes the resulting machine code and produces an object file. 
To speed up compilation, different files of the same project might be compiled in parallel.
3)  Then, the [Linker](https://en.wikipedia.org/wiki/Linker_(computing)) gets object files.
The result of the linker is an executable `.exe` file.


To speed up the build of compiled languages, [ccache](https://ccache.dev)
and [sccache](https://github.com/mozilla/sccache) are used.    
`ccache` uses the hash algorithm for the hashing of code at certain stages of the build.
`ccache` uses the hash to save a code in the cache.
When compiling a file, its hash is calculated. 
If the file is already present in the registry of compiled files, the file will not be compiled again.
Instead, the previously compiled binary file will be utilized.
This approach can significantly accelerate the build process of certain packages, reducing build times by 5-10 times.
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
Otherwise, the project will fail to build, resulting in errors.
2) `Preprocessor mode` - hash is generated based on the `.cpp` file received after the preprocessor step.


`Sccache` is similar in purpose to `ccache` but provides more functionality.
`Sccache` allows to store cached files not only locally, but also in a cloud data storage.
And `sccache` supports a wider range of languages, while `ccache` focuses on caching C and C++ compiler.


### Gradle
[Gradle](https://gradle.org) builds projects using a
[task graph](https://docs.gradle.org/current/userguide/build_lifecycle.html) that allows for synchronous execution 
of certain tasks.
`Gradle` employs
[Incremental build](https://docs.gradle.org/current/userguide/incremental_build.html#sec:how_does_it_work),
to speed up project builds.
For an incremental build to work, the tasks used to build the project must have specified
input and output files.
The provided code snippet demonstrates the implementation of a custom task in Gradle,
showcasing how inputs and outputs are specified to enable `Incremental build`:
```
task myTask {
    inputs.file 'src/main/java/MyTask.somebody' // Specify the input file
    outputs.file 'build/classes/java/main/MyTask.somebody' // Specify the output file
    
    doLast {
        // Task actions go here
        // This code will only be executed if the inputs or outputs have changed
    }
}
```


To understand how `Incremental build` works, consider the following steps:
1) Before executing a task, `Gradle` takes a 
   [fingerprint](https://en.wikipedia.org/wiki/Fingerprint_(computing))
   of the path and contents of the inputs files and saves it.
2) Then `Gradle` executes the task and saves a fingerprint of the path and contents of the output files.
3) Then, when Gradle starts a project build again, it generates a new fingerprint for the same files.
   If the new fingerprint has not changed, Gradle can safely skip this task. 
   In the opposite case, the task needs to perform an action and to rewrite outputs.
   The fingerprint is considered current if the last modification time 
   and the size of the source files have not changed.


In addition to `Incremental build`, `Gradle` also stores fingerprints of previous builds, enabling quick project builds,
for example when switching from one branch to another. This feature is known as 
the [Build Cache](https://docs.gradle.org/current/userguide/build_cache.html).


### Maven
[Maven](https://maven.apache.org) automates and manages Java-project builds.
`Maven` is based on the concept of 
[Maven LifeCycles](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html), 
which include default, clean, and site lifecycles. 
Each lifecycle consists of `phases` and these `phases` consist of sets of `goals`.

In Maven, there are default phases and goals for building any projects:

<p align="center">
  <img src="/images/defaultPhaseMaven.svg">
</p>

By default, the `phases` in Maven are inherently connected within the build lifecycle.
Each `phase` represents a specific task, and the execution order of `goals` within `phases` is determined 
by the default Maven lifecycle bindings. This means that while each `phase` operates as a series of individual tasks
and their execution order is predefined by Maven.


`Maven` utilizes caching mechanisms through the `takari-lifecycle-plugin` and `maven-build-cache-extension`:
* The [takari-lifecycle-plugin](http://takari.io/book/40-lifecycle.html) is an alternative to the default Maven lifecycle 
(building JAR files). Its distinctive feature is the use of a single universal plugin with the same functionality 
as five separate plugins for the standard lifecycle, but with significantly fewer dependencies. As a result,
it provides a much faster startup, more optimal operation, and lower resource consumption.
This leads to a significant increase in performance when compiling complex projects with a large number of modules.

* The [maven-build-cache-extension](https://maven.apache.org/extensions/maven-build-cache-extension/)
is used for large Maven projects that have a significant number of small `modules`.
A `module` refers to a subproject within a larger project.
Each `module` has its own `pom.xm` file, and there is an aggregator `pom.xml` that consolidates all the `modules`.
This plugin takes a key for a `module`, it encapsulates the essential aspects of the `module`,
including the source code and the configuration of the plugins used within it.
`Modules` with the same key are current or unchanged and the cache can efficiently restore them.
Conversely, the cache seamlessly delegates the build work to the standard Maven core,
without interfering with the build execution logic.
`maven-build-cache-extension` ensures that only the changed `modules` within the project will rebuild.


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
These tasks happen one after the other, and each task relies on the output of the one before it.
Each task has directories for input and output data, as well as a directory for storing cached data.
Using the program name, each task can receive and store data.


The previous caching mechanism in EO made use of distinct interfaces, specifically `Footprint` and `Optimization`.
These caching interfaces shared similar logic, but with minor differences.
For instance, `Footprint` verifies the EO version of the compiler, whereas the remaining checks are identical.
Additionally, the conditions for searching data in the cache had errors. 
The cached file is considered valid if the end time of goal's execution
and the time of saving goal's result to the cache are equal.
Due to this issue, the program behaved incorrectly, because saving the goal's result to the cache is not instantaneous.
After conducting an in-depth analysis of the project's incorrect operation,
several disadvantages of the previous caching mechanism in EO were brought to light:
* Incorrect search conditions for data in the cache.
* The verification method requires reading the file content, which results in inefficiencies.
* The presence of multiple caching mechanisms creates challenges in identifying and rectifying caching errors.
* Employing multiple caching mechanisms for similar entities is a suboptimal practice, 
leading to redundancy and complicating the caching infrastructure.


To address caching challenges in EO, we closely examined existing caching systems.
Maven's caching mechanisms operate at the level of `phases` and individual project modules.
However, we require a caching mechanism at the level of `goals`.
Consequently, the existing caching systems in Maven do not align with our requirements for resolving present issues.
Furthermore, we cannot use the `ccache` as the basis for creating caching in EO because `ccache` is a high-level tool
and cannot work with individual compilation tasks.
The concept of `Gradle Incremental build` bears resemblance to a tool that is essential for our purposes.
It has the capability to manage separate compilation tasks based on inputs and outputs.
However, an incremental build in Gradle may be redundant for the EO.
In contrast to other programming languages, EO currently lacks pre-existing libraries that can be integrated 
into the project.
Consequently, there is no need to generate a fingerprint for each task's data. 
Instead, it suffices to verify the last modification time of the files involved in EO compilation.
The modification time of the preceding task must not exceed that of the subsequent one.
As each task possesses directories for input and output data, accessing the desired file 
via an absolute path enables retrieval of essential information, as file name and last modified time,
from the file attributes without reading the file context.


### Conclusion
Summarizing the work completed, we can outline the following:
1) We highlighted the main problems of the current caching mechanism in EO. 
The problems stem from both the logic of the code and its architecture. 
2) We examined existing project build systems and realized that the EO language is much simpler
than existing programming languages.
This realization led us to conclude that existing caching methods are redundant for EO.
3) We proposed ideas to solve problems in the current caching implementation.















