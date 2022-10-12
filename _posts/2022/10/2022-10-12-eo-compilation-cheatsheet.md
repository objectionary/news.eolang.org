---
layout: post
date: 2022-10-12
title: "EO compilation cheatsheet"
author: mximp
---

There are still quite a lot of questions arise regarding details of EO compilation process.
Again good start point to explore it is [this blog post](https://www.yegor256.com/2021/10/21/objectionary.html).
Also I would like to share small summary of its steps key integration points and resouces. 
Please enjoy cheatsheet below. Hope you will find it handy to better understand what is happening under the hood.

<!--more-->
## EO compilation prcess integrations

### Key compilation resources

| Resource | Description |
|---|---|
| FOREIGN CATALOG | By default located at `target/eo/foreign.csv`. Contains various data for EO programs being compiled |
| PLACED CATALOG | By default located at `target/eo/placed.csv` |
| LOCAL CACHE | By default located at `<user home>/.eo/`. Used for caching files for different stages of compilation process. Persists between compilation runs. |
| XSLT SET | Various XSLTs (`.xsl` files) applied during compilation process. Located within sources: `eo-maven-plugin/src/main/resources/org/eolang/maven/pre/`, `eo-parser/src/main/resources/org/eolang/parser/` |
| OBJECTIONARY | Central storage for EO objects: <https://raw.githubusercontent.com/objectionary/home/> |
| MAVEN CENTRAL | Cetral Maven repository (defined in Maven settings). Normally: <https://repo.maven.apache.org/maven2> |

### REGISTER step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | write | Add all found EO sources. Set `id`, `eo` and `version='0.0.0'` attributes |
| `src/main/eo/` | read | Search for `.eo` sources |

### PARSE step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Program attributes `eo`, `hash` |
| FOREIGN CATALOG | write | `xmir` program attribute locating parsed program |
| `<LOCAL CACHE>/parsed/` | write | Optional: depend on `hash` presence. `xmir` files containing parsed programs |
| `<LOCAL CACHE>/parsed/` | read  | Optional: depend on `hash` presence. `.xmir` files containing parsed programs |
| `target/eo/01-parse/` | write | `.xmir` files containing parsed programs |

### OPTIMIZE step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Program attributes: `xmir2` |
| FOREIGN CATALOG | write | `xmir2` attribute containing path of optimized program XMIR |
| XSLT SET | read | Required tranformations to apply during optimization |
| `target/eo/02-steps/` | write | Optional (enabled by property). Interim XSLT steps as `.xml` files |
| `target/eo/03-optimize/` | write | XMIRs with the result of ptimization |

### DISCOVER step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Program attributes: `xmir2`, `discovered` |
| FOREIGN CATALOG | write | `discovered` attribute containing number of discovered objects within concrete XMIR |
| FOREIGN CATALOG | write | Add newly discovered objects setting `version="*.*.*"` and `discovered-at` to a path of source program file |

### PULL step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Program attributes: `eo`, `xmir` |
| FOREIGN CATALOG | write | `eo` attribute with path to newly pulled objects; `hash` attribute with corresponding version tag|
| OBJECTIONARY | read | Download objects sources for those which do not have them locally (i.e. external objects) |
| `<LOCAL CACHE>/pulled/` | read/write | Read cached and write pulled objects |
| `target/eo/04-pull` | write | pulled EO sources (`.eo` files) |


### RESOLVE step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Program attributes: `xmir`, `version`, `jar` |
| FOREIGN CATALOG | write | `jar` attribute with maven coordinates of corresponding `rt` meta; `hash` attribute with corresponding version tag|
| MAVEN CENTRAL | read | Download maven artifact for corresponding dependency (calculated from `rt` meta within EO file) |
| `target/eo/06-resolve/` | write | Unpacked `.class` files and EO sources from downloaded JAR artifacts |

### MARK step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | write | Register new programs from resolved dependencies; set `version` attribute with value taken from resolved dependency |
| `target/eo/06-resolve/` | read | Search for all `EO-SOURCES`|

### PLACE step

| Resource | Action | Description |
|---|:---:|---|
| PLACED CATALOG | read  | Search for existing entries with `kind="jar"` |
| PLACED CATALOG | write | Add new dependency entries with `kind="jar"` |
| PLACED CATALOG | write | Add identified classes and other runtime files with `id` set to path, `kind="class"`, `hash` set to file hash, `related` set to dependency set to corresponding dependency path (e.g. `org.eolang/eo-math/-/0.2.3`) |
| `target/eo/06-resolve/` | read | Search for dependencies (e.g. `org.eolang/eo-collections/-/0.0.4/`) |
| `target/classes/` | write | Copy runtime files from `06-resolved` folder |

### TRANSPILE step

| Resource | Action | Description |
|---|:---:|---|
| FOREIGN CATALOG | read  | Read attributes `xmir2`, `eo` for all programs |
| `target/eo/06-transpile/` | write | Transpiled EO programs as XMIR files |
| `target/eo/05-pre/` | write | Interim results of transpilation steps as `.xml` files |
| `target/generated-sources/` | write | `.java` files extracted from transpiled XMIRs |
| XSLT SET (`eo-maven-plugin/src/main/resources/org/eolang/maven/pre/`) | read | Required tranformations to apply during transpilation |

### COPY step

| Resource | Action | Description |
|---|:---:|---|
| `src/main/eo/` | read | EO project sources to copy |
| `target/classes/EO-SOURCES/` | write | EO project sources |

### UNPLACE step

| Resource | Action | Description |
|---|:---:|---|
| PLACED CATALOG | read | Get class entries previously placed |
| PLACED CATALOG | write | Set `unplaced` flag |
| `target/classes/` | write | Delete previously placed files |

### UNSPILE step

| Resource | Action | Description |
|---|:---:|---|
| `target/classes/` | read | Iterate over `.class` files  |
| `target/classes/` | write | Delete those `.class` files for which `.java` counterpart exist |
| `target/generated-sources/` | read | Check `.java` files for presence |

