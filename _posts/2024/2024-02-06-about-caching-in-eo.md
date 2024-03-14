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
for EO version `0.34.0`. The error occurred because the algorithm compared
the compilation time and caching time to search for the needed file.
This is not the most reliable verification method,
because caching time does not have to be equal to compilation time.
That we came to the conclusion that we need caching with a reliable verification method
that does not require reading a file system.

The goal of this blog is to research caching in frequently used build systems (`ccache`, `Maven`, `Gradle`)
and to create effective caching in [EO](https://github.com/objectionary/eo).

<!--more-->

## Build caching of existing build systems

### ccache/sccache
In compiled programming languages, building a project takes a long time.
The reason for the lengthy compilation time is that time is spent on preparing,
optimizing, checking the code, and so on.
To speed up the assembly of compiled languages, ccache and sccache are used.
Let's look at the assembly scheme using C++ as an example
to imagine the build process in compiled languages:

<p align="center">
  <img src="/images/ccache.svg">
</p>

1) First, preprocessor gets the input files. The input files are code files and header files.
The preprocessor removes comments from the code and converts the code in accordance
with macros and executes other directives, starting with the “#” symbol
(such as #include, #define, various directives like #pragma).
The result is a single edited file with human-readable code that the compiler will get.


2) The compiler receives the finished code file and converts it into machine code, presented in an object file.
At the compilation stage, parsing occurs, which checks whether the code matches
rules of a specific programming language. Next, the parsing occurs preprocessor code into machine code
according to the rules.
At the end of its work, the compiler optimizes the resulting machine code and produces an object file. 
To speed up compilation, different files of the same project are compiled in parallel,
that is, we receive several object files at once.

3)  After all received project object files are passed to the linker.
Linker is a program that combines program components, written in assembly language or a high-level programming language,
into an executable file or library. The result of the linker is an executable .exe file.


As a result, in compiled languages, multiple files are simultaneously and independently converted
into machine code at the compilation stage.
This machine code is then combined into one executable file.


`ccache` has two main caching methods:
1) `Direct mode` - hashcode is generated based on the source code.
2) `Preprocessor mode` - hashcode is generated based on the result of preprocessor.

The hashcode includes information: file contents, directory, compiler information, compilation time, extensions
used by the compiler. A compressed machine code file is placed in the cache using the received key.

`Direct mode` compiles the program faster, since the preprocessor step is skipped. 
BuHowever,the header files are not checked for changes, so the wrong project may be built.
`Preprocessor mode` is slower than `direct mode`, but the right project is built always.

Sccache, unlike ccache, allows the cache to be stored not only locally but also in the cloud,
and it also has fixed some bugs (for example, there is a check of header files, which makes direct mode more accurate).


### Maven
`Maven` automates and manages Java-project builds. Building a project in `Maven` is completed in three
maven [LifeCycles Maven](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html),
which consist of `phases`. `Phases` in turn consist of sets of `goals`.

`Maven` has default `phases` and `goals`  for building any projects:

<p align="center">
  <img src="/images/defaultPhaseMaven.svg">
</p>

In `Maven` all phases and goals are executed strictly in order, linearly.
But in `Maven` there is no build-time caching as such.
`Maven` suggests rebuilding only changed project modules to speed up the build process.

### Gradle
But unlike `Maven`, `Gradle` builds projects using a task graph - 
[Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph),
in which some tasks can be executed synchronously.
To speed up project builds, `Gradle` employs incremental builds
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
Before executing a task, `Gradle` makes a fingerprint of the path
and contents of the source files and saves it.
If the task completes successfully, `Gradle` also makes a fingerprint from the resulting files.
To avoid re-fingerprinting the original files, `Gradle` checks the last modification time and the size of the original
files before reassembling. This allows `Gradle` to use the results already obtained when the project is rebuilt.
Additionally, `Gradle` stores fingerprints of previous builds enabling quick project builds,
for example when switching from one branch to another - known as the - `Build Cache`.




### EO build cache

EO code uses the `Maven` build system to assembly.
For this purpose, the `eo-maven-plugin` plugin was created,
which contains the necessary goals for working with EO code.
As mentioned earlier, the assembly of projects in `Maven` occurs in a specific order of phases.
In the diagram you can observe the main phases and their goals for the EO last version of the compiler:

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
caching at each step to speed up the assembly of the EO program.

In EO version `0.34.0`, 
caching used unrelated `Footprint` and `Optimization` interfaces for different `Mojo`,
within which mostly the same methods were used.
The difference between interfaces is that `Footprint` checks the EO version of the compiler,
while the rest of the checks are exactly the same.


Now, goals are `ParseMojo`, `OptimazeMojo` и `ShakeMojo` , in which caching can be applied,
have directory of results and directory of cache.


The disadvantages of initial caching in EO include:
* The compilation time and the time of saving to the cache must be equal, which can be challenging to verify.
* Verification data is read from a file on disk, which is a long and expensive operation.
* Each purpose uses its own classes and interfaces for data caching, making the code difficult to extend and read.



To address these disadvantages, the following solutions are proposed:


1) Create a new class `Cache` responsible for data verification, saving to cache and loading from cache.

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


`List<CacheValidation>` is a list of validations. Validations implemented from the `CacheValidation` interface.
Different `Mojo` can use different validations.


```
public interface CacheValidation {
    boolean validate(final Path source, final Path cache) throws IOException;
}
```

2) To avoid reading from disk, we will use file paths `Path`.
The classes `Path` and `Files` have methods to obtain the necessary information.


3) The relevance of the cached data will be checked by the condition
that the time of the last modification of the source file must be earlier than or equal to that saved in the cache.

These solutions will speed up compilation in the build system `Maven`.


### Conclusion
There is an EO program `program.eo`, which is launched for the first time.
At each `Mojo` stage, the execution results will be saved to the cache of the current `Mojo`.
If this program is run again, these `Mojo` will receive data from the cache,
without wasting time and computer resources on recompilation.
If we change something in the `program.eo` file, the program will have to be recompiled,
since the last modification time the original file will be later than those stored in the cache.
As a result of `Mojo` work, the cache was overwritten.

















