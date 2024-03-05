---
layout: post
date: 2024-02-06
title: "Build cache in EO and other build systems"
author: Alekseeva Yana
---

<!--more-->

## Introduction
Wasting a lot of time on building a project is a programming problem. At the moment a programmer starts an
assembly, he loses focus on a task and spends valuable working time. Different build systems use many tools,
helping to assemble a project faster, namely caching, task parallelization, distributed building and much more.
The subject of this article is caching, because completed tasks caching allows not to spend resources again. 
So in [EO](https://github.com/objectionary/eo) caching is used for speeding up programs work.
While developing [EO](https://github.com/objectionary/eo) we found caching errors in `eo-maven-plugin`
for EO version `0.34.0`. The error occurred, because using a file name and comparing equality of
compilation time and caching time is not the most reliable verification. Unit tests were written showing that 
cache does not work correctly. Also reading a file was necessary for getting a programme name 
that slowed down an assembly. 
That we came to conclusion that we need caching with a reliable verification which does not require reading a file
from disk. And using cache should save us enough time for building a project.

The goal of this article is to research caching in frequently used build systems (`ccache`, `Maven`, `Gradle`)
and to create effective caching in [EO](https://github.com/objectionary/eo).

## Build caching of existing build systems

### ccache/sccache
In compiled programming languages, building a project takes a long time.
The reason of long compilation is time is spent on preparing, optimizing and checking the code, and so on.
To speed up the assembly of compiled languages, ccache and sccache are used.
Let's look at the compilation scheme using C++ as an example,
to imagine the build process in compiled languages:


![Picture 1](/images/ccach.svg)

1) First, preprocessor gets the input files. Input files are code files and header files.
The preprocessor removes comments from the code and converts the code into in accordance
with macros and executes other directives, starting with the “#” symbol
(such as #include, #define, various directives like #pragma).
The result is a single edited file with human-readable code that can be submitted to the compiler.


2) The compiler receives the finished code file and converts it into machine code, presented in an object file.
At the compilation stage, parsing occurs, which checks whether the code matches
rules of a specific programming language. Next, the code is parsed into machine code according to the rules.
At the end of its work, the compiler optimizes the resulting machine code and produces an object file. 
To speed up compilation, different files of the same project are compiled in parallel,
that is, we receive several object files at once.

3)  After all received project object files are passed to the linker.
Linker is a program that combines program components, written in assembly language or a high-level programming language,
to an executable file or library. The result of the linker is an executable .exe file.


As a result, in compiled languages, multiple files are simultaneously and independently converted into machine code at the compilation stage.
This machine code is then combined into one executable file.


`ccache` has two main caching methods они:
1) `Direct mode` - hashcode is generated based on the source code.
2) `Preprocessor mode` - hashcode is generated based on the result of preprocessor.

The hashcode includes information: file contents, directory, compiler information, compilation time, extensions
used by the compiler. A compressed machine code file is placed in the cache using the received key.

`Direct mode` compiles the program faster, since the preprocessor step is skipped. 
But header files are not checked for changes, so the wrong project may be built.
`Preprocessor mode` is slower than `direct mode`, but right project is built always.

Sccache, unlike ccache, allows to store the cache not only locally but also in the cloud,
and it also has fixed some bugs (for example, there is a check of header files, which makes direct mode more accurate).


### Maven
`Maven` automates and manages Java-projects build. Building a project in `Maven` is completed in three
maven [LifeCycles Maven](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html),
which consist of `phases`. `Phases` in turn consist of sets of `goals`.

`Maven` has default `phases` and `goals`  which build any projects.


![Picture 2](/Users/yanaalekseeva/IdeaProjects/news.eolang.org/images/defaultPhaseMaven.svg)


In `Maven` all phases and goals are executed strictly in order, linearly.
But in `Maven` there is no build-time caching as such.
`Maven` suggests rebuilding only changed project modules to speed up the build process.

### Gradle
`Gradle`, like `Maven`, builds a project in 
[LifeCycles Gradle](https://docs.gradle.org/current/userguide/build_lifecycle.html), which consists of phases.
But unlike `Maven`, `Gradle` builds projects using a task graph - 
[Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph),
in which some tasks can be executed synchronously.
To speed up project builds, `Gradle` uses incremental builds
[Incremental build](https://docs.gradle.org/current/userguide/incremental_build.html#sec:how_does_it_work).
For an incremental build to work, the tasks that are used to build the project must have
source and output files must be specified.
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
Every time before executing a task, `Gradle` makes a fingerprint of the path
and contents of the source files and saves it.
If the task completes successfully, then `Gradle` also makes a fingerprint from the resulting files.
To avoid re-fingerprinting the original files, `Gradle` checks the last modification time and the size of the original
files before reassembling. Thus, when the project is rebuilt, some or all of the tasks may be
not completed, but to use the results already obtained.
`Gradle` also stores fingerprints of previous builds so that projects can be built quickly, for example when switching
from one branch to another - `Build Cache`.




### EO build cache

EO code is compiled using the `Maven` build system.
For this purpose, the `eo-maven-plugin` plugin was written,
which contains the goals necessary for working with EO code.
As was written above, the assembly of projects in `Maven` occurs in a certain order of phases.
In the diagram you can see the main phases and their goals for the EO version of the compiler (specify version):

![Picture 3](/images/EO.svg)

In [Picture 3](/images/EO.svg) the goals from the `eo-maven-plugin`
are highlighted in green.


But the actual work with EO code takes place in `AssembleMojo`.
`AssembleMojo` is the goal consisting of other goals that work with the EO file 
[Picture 4](/images/AssembleMojo.svg).

![Picture 4](/images/AssembleMojo.svg)

Each goal in `AssembleMojo` is a specific compilation step for EO code, and we need to use
caching at each step to speed up the assembly of the EO program.

In EO version `0.34.0`, 
caching for different `Mojo` was done using unrelated different `Footprint` and `Optimization` interfaces,
within which mostly the same methods were used.
The difference between interfaces is that in `Footprint` the EO version of the compiler is checked,
while the rest of the checks are exactly the same.


Now goals are `ParseMojo`, `OptimazeMojo` и `ShakeMojo` , in which caching can be applied,
have directory of results and directory of cache.


The disadvantages of initial caching in EO:
* the compilation time and the time of saving to the cache must be equal. 
The problem with this verification is that the moment of compilation and the moment of saving to the cache must coincide.
* verification data is read from a file on disk. This is a long and expensive operation.
* each purpose uses its own classes and interfaces for data caching. 
This makes the code difficult to extensibility and readability.


Therefore, our target is to create a single class responsible for caching data
and loading the necessary data from the cache, which can be used for any `Mojo` from the `eo-maven-plugin`.


How do we want to fix this disadvantages:
1) Create a new class `Cache` that will be responsible for data verification, saving to cache and loading from cache.

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


`List<CacheValidation>` is a list of validations that are implemented from the `CacheValidation` interface.
Different validations can be applied for different `Mojo`.


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

















