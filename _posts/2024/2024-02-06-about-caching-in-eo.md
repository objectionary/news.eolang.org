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
To achieve the goal, we will briefly look at how frequently used build systems (such as ccache, Maven, Gradle)
in order to gain a deeper understanding of the caching concepts employed within them and to development caching in EO.

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
2) The compiler receives the file `.cpp` from the preprocessor and converts it into object file - `.obj`.
At the compilation stage, parsing checks whether the code matches rules of a specific programming language.
At the end, the compiler optimizes the resulting machine code and produces an object file. 
To speed up compilation, different files of the same project might be compiled in parallel.
3)  Then, the [Linker](https://en.wikipedia.org/wiki/Linker_(computing)) gets object files.
The result of the linker is an executable `.exe` file.


To speed up the build of compiled languages, [ccache](https://ccache.dev)
and [sccache](https://github.com/mozilla/sccache) are used.    
`ccache` uses the hash algorithm for the hashing of code at certain stages of the build.
`ccache` uses the hash to save a code in the cache. 
The [`ccache` hash](https://ccache.dev/manual/4.8.2.html#_common_hashed_information) is
based on:
* the file contents
* the current directory of the file
* the name of the compiler
* the compiler’s size and modification time
* extensions used by the compiler.

Moreover, `ccache` has two types of the hashing:
1) `Direct mode` - the hash is generated based on the source code only. 
This mode allows to build the program faster, since the preprocessor step is skipped.
When using this mode, the user must be sure that the external libraries, using in a project, have not changed.
Otherwise, the project will build with errors.
2) `Preprocessor mode` - hash is generated based on the `.cpp` file received after the preprocessor step.
`Preprocessor mode` is slower than `direct mode`, but the project is built without errors.

`Sccache`, unlike `ccache`, allows you to store cached files not only locally, but also in a cloud data storage.
And `sccache` supports a wider range of languages, while `ccache` focuses on caching C and C++ compiler.


### Gradle
[Gradle](https://gradle.org) builds projects using a
[task graph](https://docs.gradle.org/current/userguide/build_lifecycle.html) that allows for synchronous execution 
of certain tasks.
`Gradle` employs
[Incremental build](https://docs.gradle.org/current/userguide/incremental_build.html#sec:how_does_it_work),
to speed up project builds.
For an incremental build to work, the tasks used to build the project must have specified
source and output files.
The provided code snippet demonstrates the implementation of a custom task in Gradle,
showcasing how inputs and outputs are specified to enable `Incremental build`:
```
task myTask {
    inputs.dir 'src/main/java/MyTask.somebody' // Specify the input directory
    outputs.dir 'build/classes/java/main/MyTask.somebody' // Specify the output directory

    doLast {
        // Task actions go here
        // This code will only be executed if the inputs or outputs have changed
    }
}
```
`Gradle` uses this information to determine if a task is up-to-date and needs to perform any work.

To understand how `Incremental build` works, consider the following steps:
1) Before executing a task for the first time, `Gradle` takes a 
   [fingerprint](https://en.wikipedia.org/wiki/Fingerprint_(computing))
   of the path and contents of the source files and saves it.
2) Then `Gradle` executes the task and saves a fingerprint of the path and contents of the output files.
3) Before each rebuilding of the task, `Gradle` generates a new fingerprint of the source files
   and compares it with the current fingerprint. 
   The fingerprint is considered current if the last modification time 
   and the size of the source files have not changed. 
   If none of the inputs or outputs have changed, Gradle can skip that task.


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
by the default Maven lifecycle bindings. This means that while each `phase` operates as a series of individual tasks,
they are part of a cohesive build lifecycle, and their execution order is predefined by Maven.


`Maven` supports `Incremental build` through plugins the `takari-lifecycle-plugin` and 
`maven-build-cache-extension`.
The [takari-lifecycle-plugin](http://takari.io/book/40-lifecycle.html) is an alternative to the default Maven lifecycle 
(building JAR files). Its distinctive feature is the use of a single universal plugin with the same functionality 
as five separate plugins for the standard lifecycle, but with significantly fewer dependencies. As a result,
it provides a much faster startup, more optimal operation, and lower resource consumption.
This leads to a significant increase in performance when compiling complex projects with a large number of modules.

The [maven-build-cache-extension](https://maven.apache.org/extensions/maven-build-cache-extension/)
is used for large Maven projects that have a significant number of small modules. 
This plugin takes a key for a project module, it encapsulates the essential aspects of the module,
including the source code and the configuration of the plugins used within it.
Projects with the same key are considered current (unchanged) and can be efficiently restored from the cache.
Conversely, projects that generate different keys are deemed outdated (changed),
prompting the cache to initiate a complete rebuild for them. In the event of a cache miss,
where an outdated project requires a complete rebuild,
the cache seamlessly delegates the build work to the standard Maven core,
without interfering with the build execution logic.
This ensures that only the changed modules within the project are rebuilt,
minimizing unnecessary overhead and optimizing the build process.


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
To speed up the EO program rebuild process, it is helpful to save the results of each goal.
This avoids repeating actions and makes the compilation more efficient.
Using caching methods significantly speeds up the build process.


In this chapter, we introduce the keywords:
* `the source file`: This file serves as the input for goal operations.
* `the cached file`: This file contains the results of goal's execution.


The previous caching mechanism in EO made use of distinct interfaces, specifically `Footprint` and `Optimization`,
both of which derive from the `SafeMojo` class.
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


To address these disadvantages, the following solutions are proposed:
1) Creating a unified caching mechanism for all goals associated in EO code compilation.
This mechanism, represented by the `Cache` class, will assume responsibility for data validation,
cache storage, and retrieval.
To improve the flexibility for different data verification conditions, 
the constructor of the `Cache` class  will accept a list of validations.
Here's the corresponding code:

```
public class Cache {

    private List<CacheValidation> validations;
    
    public Cache(final List<CacheValidation> cv) {
        this.validations = cv;
    }
    
    public Optional<XML> load(final Path source, final Path cache) {...};
    
    public void save(final Path cache, final Scalar<String> program, final Path relative) {...};
}
```

The `List<CacheValidation>` represents a list of validations implemented from the `CacheValidation` interface.
This interface defines the structure for validations within the `Cache` class.
The `CacheValidation` interface has the only method ensuring that each validation contains a specific test condition.

```
public interface CacheValidation {
    boolean validate(final Path source, final Path cache) throws IOException;
}
```

2) In order to minimize disk access, we will utilize file paths represented by the `Path` class.
By leveraging methods provided by the `Path` and `Files` classes,
we can obtain essential information such as the file name and the time of the last modification.
The file name plays a crucial role in locating the cached file within the directory,
while the time of the last modification enables us to determine whether
the source file is older or equal in age to the cached file.
Given that the project build process in Maven is linear, 
these conditions are deemed sufficient for our caching mechanism.


3) Searching for a cached data will use the following conditions:
  * `The source file` and `the cached file` should have same file name;
  * Each goal involved caching should have both a cache directory and a directory of result files. 
    The directory of result files corresponds to the directory of source files for the subsequent goal.
  * The time of the last modification of the source file should be earlier or equal than cached file.


Example: Let's consider an EO program named `program.eo`, which is executed for the first time.
The cache of each goal will save the execution results in the cache directory and the result directory.
When this program is run again without changes, these goal will receive data from the cache,
without executing of task and rewriting of result.
However, if we make changes to the `program.eo` file or `the source files` of the goals and execute again,
the execution result of goals was overwritten in the directory of result files and the cache directory.
This approach effectively protects the program from artificial changes during the build process.


### Conclusion
In this article, we explored various build systems and their caching methods.
We were motivated to find an efficient caching approach for EO due to issues discovered during bug investigation.
The previous caching mechanism was flawed logically and architecturally, making it ineffective. 
As a result, we discussed the problems, suggest solutions, and outline the criteria 
for implementing a new caching system in EO.















