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
contains a comparison of the compilation time and caching time to search for the cached file.
This is not the most reliable verification method,
because caching time does not have to be equal to compilation time.
We came to the conclusion that we need caching with a reliable verification method.
And this verification method should not use the information that the cached file contains.

The goal of this blog is to research caching in frequently used build systems (`ccache`, `Maven`, `Gradle`)
and to implement effective caching in [EO](https://github.com/objectionary/eo).

<!--more-->

## Build caching of existing build systems

### ccache/sccache
In compiled programming languages, building a project containing many source code files takes a long time.
This time is spent on loading of libraries, preparing, optimizing, checking the code, and so on.
To speed up the assembly of compiled languages, [ccache](https://ccache.dev) 
and [sccache](https://github.com/mozilla/sccache) are used.
Let's look at the assembly scheme using C++ as an example:

<p align="center">
  <img src="/images/defaultCPhase.svg">
</p>

1) First, preprocessor retrieves the source code files,
which consist of both source files `.cpp` and header files `.h`.
The result is a single file `.cpp` with human-readable code that the compiler will get.
2) The compiler receives the edited code file `.cpp` and converts it into object file - `.obj`.
At the compilation stage, parsing checks whether the code matches rules of a specific programming language.
At the end, the compiler optimizes the resulting machine code and produces an object file. 
To speed up compilation, different files of the same project might be compiled in parallel.
3)  Then, the [Linker](https://en.wikipedia.org/wiki/Linker_(computing)) gets object files.
The result of the linker is an executable `.exe` file.

    
`ccache` has hash algorithm, for the hashing of information to find cached files fast. 
The [`ccache` hash](https://ccache.dev/manual/4.8.2.html#_common_hashed_information)
includes information:
* the file contents
* the current directory of the file
* the name of the compiler
* the compiler’s size and modification time
* extensions used by the compiler. 


A compressed machine code file is placed in the cache using the received key.


`ccache` has two main caching methods:
1) `Direct mode` - hash is generated based on the source code. 
`Direct mode` allows to build the program faster, since the preprocessor step is skipped.
However, header files are not checked for changes, so the project may be built with not verified header files.
2) `Preprocessor mode` - hash is generated based on the `.cpp` file received after the preprocessor step.
`Preprocessor mode` is slower than `direct mode`, but the project is built with verified header files.

`Sccache`, unlike `ccache`, allows you to store cached files not only locally, but also in a cloud data storage.
And `sccache` supports a wider range of languages, while ccache focuses on caching C and C++ compiler.


### Maven
[Maven](https://maven.apache.org) automates and manages Java-project builds. 
Building a project in `Maven` is completed in three
[Maven LifeCycles](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html),
which consist of `phases`. `Phases` consist of sets of `goals`.

`Maven` has default `phases` and `goals`  for building any projects:

<p align="center">
  <img src="/images/defaultPhaseMaven.svg">
</p>

In `Maven` all phases and goals are executed strictly in order, linearly.
`Maven` uses added extensions from Gradle for caching.
`Maven` suggests rebuilding only changed project modules to speed up the build process.

### Gradle
But unlike `Maven`, [Gradle](https://gradle.org) builds projects using a task graph - 
[Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph),
in which some tasks can be executed synchronously.
To speed up project builds, `Gradle` employs
[Incremental build](https://docs.gradle.org/current/userguide/incremental_build.html#sec:how_does_it_work).
For an incremental build to work, the tasks used to build the project must have specified
source and output files.
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

How work `Incremental build`:
1) Before executing a task, `Gradle` creates a [fingerprint](https://en.wikipedia.org/wiki/Fingerprint_(computing))
   of the path and contents of the source files and saves it.
2) Then `Gradle` executes the task and saves a fingerprint of the path and contents of the output files.
3) Before each rebuilding of the task, `Gradle` generates a fingerprint of the source files
   and compares it with the current fingerprint. 
   The fingerprint is considered current if the last modification time 
   and the size of the source files have not changed. 
   If none of the inputs or outputs have changed, Gradle can skip that task.



Additionally, `Gradle` stores fingerprints of previous builds enabling quick project builds,
for example when switching from one branch to another - known as the -
[Build Cache](https://docs.gradle.org/current/userguide/build_cache.html).



### EO build cache

EO code uses the `Maven` build system to build.
For this purpose, the `eo-maven-plugin` plugin was created,
which contains the necessary goals for working with EO code.
As mentioned earlier, the build of projects in `Maven` occurs in a specific order of phases.
In the diagram you can observe the main phases and their goals for the EO:

<p align="center">
  <img src="/images/EO.svg">
</p>

In [Picture 3](/images/EO.svg) the goals from the `eo-maven-plugin`
are highlighted in green.


However, the actual work with EO code takes place in `AssembleMojo`.
`AssembleMojo` is the goal consisting of other goals that work with the EO file, as shown in
[Picture 4](/images/AssembleMojo.svg).


<p align="center">
  <img src="/images/AssembleMojo.svg">
</p>

Each goal in `AssembleMojo` is a specific compilation step for EO code, and we need to use
caching at each step to speed up the build of the EO program.


In EO version `0.34.0`, 
caching used unrelated `Footprint` and `Optimization` interfaces for different `Mojo`,
which used the same methods.
The difference between interfaces is that `Footprint` checks the EO version of the compiler,
while the rest of the checks are exactly the same.

In this chapter ` the source file` - is a file, which `Mojo` receives, 
`the cached file` - is a file with the result of executing `Mojo`,
`the program file` - is initial EO program file (`program.eo`).


The disadvantages of initial caching in EO include:
* The cached file is actual if the compilation time and the time of saving to the cache are equal.
* Verification data is read from a file on file system.
* Each goal uses own classes and interfaces for data caching, making the code difficult to extend and read.


To address these disadvantages, the following solutions are proposed:
1) Create a new class `Cache` responsible for data verification, saving to cache and loading from cache.
Since each `Mojo` that involves caching has directories for saving and caching results, 
we just need to create a class responsible for saving and loading their cache data.
Each `Mojo` will have the own class `Cache` with own a list of validations.
If all `Mojos` have the same validations, then one `Cache` is enough.

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

`List<CacheValidation>` is a list of validations. Validations are implemented from the `CacheValidation` interface.
The `CacheValidation` interface has the only method, that must contain one test condition.
```
public interface CacheValidation {
    boolean validate(final Path source, final Path cache) throws IOException;
}
```

2) To avoid reading from disk, we will use file paths `Path`.
The classes `Path` and `Files` have methods to obtain the necessary information - the file name
and the time of the last modification. The file name is necessary to find the cached file in the directory.
The time of the last modification is necessary to check the source file is older(or equal) than the cached file.
These conditions should be enough for us, since the build of projects in Maven is linear.


3) Searching for a cached data will use the following conditions:
  * The source file and cached file should have same file name;
  * Each `Mojo`, involved caching, should have a cache directory and a directory of result files. 
    The directory of result files is directory of source files for next `Mojo`.
  * The time of the last modification of the source file should be earlier or equal than cached file.


Example: there is an EO program `program.eo`, which is launched for the first time;
the cache of each `Mojo` will save the execution results in cache directory and result directory;
when this program is run again, these `Mojo` will receive data from the cache,
without executing of task and rewriting of result.
If we change something in the `program.eo` file or the source files,
the program or will have to be executed again or the execution result of `Mojo` was overwritten.
This way the program will be protected from artificial changes during the build process.


### Conclusion
In this blog, we showed that `Maven` builds the EO code using the goals of the `eo-maven-plugin`.
Since the Maven goals work in a strict order and linearly,
we only need to check that the last modification time of the source files is not younger than the cached files.
The cached file and the source file should have the same name 
(but not the same file format, for example - name.eo and name.xml).
This condition is necessary so that you can quickly find the cached file in the file system.
Each Mojo participating in caching should have its own cache directory.

















