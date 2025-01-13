---
title: "Gantry"
date: 2020-09-26T20:00:00+02:00
author: "Axel Lehmann"
authorAvatar: "img/project_gantry/lehmann.png"
tags: [gantry, wharfer, docker, docker-compose]
categories: [project]
image: "img/project_gantry/independent-pipelines-matrix.png"
draft: false
---
Gantry allows [`docker-compose`](https://docs.docker.com/compose/)-like deployments with [`wharfer`](https://github.com/ad-freiburg/wharfer).
`wharfer` is a replacement for the `docker` executable used and maintained by the Chair of Algorithms and Data Structures.
Additionally, `gantry` allows executing containers in a sequential order.
Within this sequential order, `gantry` tries to execute as many containers in parallel as possible.
This minimizes overall execution time but keeps results deterministic.

<!--more-->

# Content
1. [Introduction](#introduction)
1. [Features](#features)
 1. [docker-compose compatibility](#docker-compose-compatibility)
 1. [Pipelines](#pipelines)
 1. [gantry.env.yml](gantry-env-yml)
 1. [Preprocessor](#preprocessor)
 1. [DOT graph](#dot-graph)
1. [Implementation](#implementation)
 1. [Environment calculation](#environment-calculation)
 1. [Pipeline generation](#pipeline-generation)
 1. [Pipeline execution](#pipeline-execution)
1. [Source code, executables, and more](#source-code-executables-and-more)

# Introduction

[`wharfer`](https://github.com/ad-freiburg/wharfer) is a drop-in replacement for the virtualization software Docker with additional security features.
To allow [`docker-compose`](https://docs.docker.com/compose/)-like deployments for wharfer we developed [`gantry`](https://github.com/ad-freiburg/gantry).

Docker deployments consist of multiple interacting containers, e.g. a database backend and a website frontend.
docker-compose starts all these containers and their dependencies at the same time.
gantry provides the same basic functionality while providing access to the security features of wharfer.
Besides this basic docker-compose compatibility, gantry introduces the concept of *blocking dependencies*.
A blocking dependency must finish execution before the dependent container starts.
This allows the creation of *data pipelines*.
The order of execution inside a pipeline is defined by these dependencies: if \\(B\\) requires \\(A\\), gantry only starts \\(B\\) after \\(A\\) has exited.

To achieve the same order of execution with docker-compose multiple solutions exists:

1. Combine \\(A\\) and \\(B\\) in a single container
1. Mount a shared volume between the containers and let \\(B\\) listen for file changes by \\(A\\).
1. Synchronize \\(A\\) and \\(B\\) by using network-based solutions.

All of these solutions require additional synchronization (shared volume (2) or port (3)) or loss of separation (combining \\(A\\) and \\(B\\) inside a single container).
In particular, (1) greatly reduces composability as \\(A\\) and \\(B\\) are now a single container \\(AB\\).
To use \\(B\\) with \\(C\\) instead of \\(A\\), \\(B\\) must be duplocated in a new container \\(CB\\).

# Features

The `docker-compose` compatibility provided by `gantry` and *Pipelines* are the main features of `gantry`.
Additional features not available in `docker-compose` are provided:

* additional global and per-container configuration through `gantry.env.yml`
* a preprocessor for `gantry.yml` (which are extended `docker-compose.yml` files)
* `gantry dot` to dump *deployments* and *pipelines* as dot-files.

## `docker-compose` compatibility
In general, a deployment consists of multiple containers which interact and are managed together.
`docker-compose` starts *all* requested containers and their dependencies at the same time.
Deployments are used to manage multiple parts of a system e.g. a database and the frontend of a website.
Most containers in a deployment provide at least one service to other containers e.g. a database storing contents of a website.
Therefore, *containers* are referred to as *services* inside the `docker-compose.yml`.
`gantry` uses the [services](https://docs.docker.com/compose/compose-file/#service-configuration-reference) keyword and is compatible with basic `docker-compose.yml` files in both [version 2](https://docs.docker.com/compose/compose-file/compose-file-v2/), and [version 3](https://docs.docker.com/compose/compose-file/).

A WordPress blog can be setup as shown in the [docker-compose example](hhttps://github.com/ad-freiburg/gantry/tree/master/examples/docker-compose) which uses the following `docker-compose.yml`:

```
version: '3.3'

services:
  db:
    image: mysql:5.7
    volumes:
      - ./data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: somewordpress
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8000:80"
    restart: always
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
```

`gantry` is not only able to use existing images, but can also be used to create new images using the [build](https://docs.docker.com/compose/compose-file/#build) keyword.
To allow for multiple container definitions multiple additional keywords are supported.
[context](https://docs.docker.com/compose/compose-file/#context) allows to build different directories.
[dockerfile](https://docs.docker.com/compose/compose-file/#dockerfile) allows to build Dockerfiles with different names.
[args](https://docs.docker.com/compose/compose-file/#args) can be used to provide build-time arguments which can be used inside Dockerfiles.

### Supported `docker-compose` keywords

In this section every keyword is linked to the official documentation for docker-compose. Everything should work as described there, exceptions are explicitly stated besides the keyword.

* [`build`](https://docs.docker.com/compose/compose-file/#build)
  * [`context`](https://docs.docker.com/compose/compose-file/#context)
  * [`dockerfile`](https://docs.docker.com/compose/compose-file/#dockerfile)
  * [`args`](https://docs.docker.com/compose/compose-file/#args)
* [`command`](https://docs.docker.com/compose/compose-file/#command) 
* [`depends_on`](https://docs.docker.com/compose/compose-file/#depends_on)
* [`entrypoint`](https://docs.docker.com/compose/compose-file/#entrypoint)
* [`image`](https://docs.docker.com/compose/compose-file/#image)
* [`ports`](https://docs.docker.com/compose/compose-file/#ports), only the short syntax is supported:

```
    ports:
      - "8000:80"
```

* [`restart`](https://docs.docker.com/compose/compose-file/#restart)
* [`volumes`](https://docs.docker.com/compose/compose-file/#volumes), only the short syntax is supported:
  
```
  volumes:
      - ./data:/var/lib/mysql
```

### Supported `docker-compose` commands

* `build`<br/>
  Builds all images
* `down`<br/>
  Stop and remove containers, and networks created by `up`
* `kill`<br/>
  Force stop containers
* `list`<br/>
  Lists all defined services and steps
* `logs`<br/>
  View output from containers.
* `rm`<br/>
  Removes stopped containers
* `pull`<br/>
  Pulls images for services/steps defined in a Compose file, but does not
  start the containers.

### Deployment

`gantry` can be used for `docker-compose`-like deployments of different scales.
Small deployments like the WordPress example in the introduction can be managed as well as deployments consisting of multiple *services*.
The [Broccoli](http://broccoli.cs.uni-freiburg.de/demos/BroccoliFreebase/), [Yago](http://broccoli.cs.uni-freiburg.de/BroccoliYago/), and [Freebase Easy](http://freebase-easy.cs.uni-freiburg.de/browse/) demos are all powered by a single `docker-compose.yml` and `gantry`.

![Broccoli Deployment](/img/project_gantry/broccoli-deployment.png)

The *proxy service* exposes the relevant ports to the host, all other services remain in the separate network without the need to expose ports on their own.
This keeps the backends from being exposed to the world and other deployments, as each deployment creates its own network.
This network isolation is a basic feature of `docker-compose`.
More complex networking, e.g. [multiple networks](https://docs.docker.com/compose/networking/#specify-custom-networks) in one deployment, is currently not supported by `gantry`.

## Pipelines

The biggest addition is the introduction of *blocking dependencies*.
A blocking dependency specifies for a given container which other containers have to finish before the execution of the given container starts.
Such a sequence of multiple containers creates a *pipeline* where data moves through them in a predefined order.
Inside the *pipeline* data is manipulated container after container i.e. step by step.

`gantry` introduces the `steps` keyword which is used on the same hierarchy level as the `services` keyword inside `docker-compose.yml` files.
Each step is required to exit before its dependents are started, thus a sequential execution order is guaranteed.
To explicitly mark this sequential nature, the `after` keyword is used in *steps* instead of the `depends_on` as in *services*.
Both lists are merged when calculating the dependencies for each *step* or *service*.

These new keywords stop `docker-compose` from executing a `docker-compose.yml`.
This breaking behavior ensures that sequential pipelines are not run in the non-sequential manner `docker-compose` would use.
To better mark a file using these keywords the name `gantry.yml` is used.

A pipeline can make use of both *services* and *steps*.
As an example, the end-to-end tests of [QLever](https://github.com/ad-freiburg/QLever), a *Super-Efficient SPARQL Search Engine* developed by the Chair of Algorithms and Data Structures, can be run using a [`gantry.yml`](https://github.com/ad-freiburg/gantry/tree/master/examples/qlever_e2e).

![QLever End-to-End Test Pipeline](/img/project_gantry/qlever-e2e.png)

Boxes represent `gantry` *steps* while the oval *qlever* represents a `docker-compose` *service*.
The *wait_for_qlever* step shows the problem which `gantry` tries to solve for sequential *steps*.
*qlever* is the backend service needed by *run_queries* and therefore, must be kept alive.
If *run_queries* would be executed at the same time it would always fail as the backend is not available immediately.
The *wait_for_qlever* step encapsulates the polling for the backend and provides a reusable way to wait for a QLever instance.
This is the only instance of explicit waiting required in the whole pipeline by using `gantry`. Using `docker-compose` explicit waiting would be necessary in each step.

As services started by `docker-compose` remain running, the option to handle the life cycle of services is introduced inside an additional `gantry.env.yml` file. 

### Parallelization

`gantry` tries to execute as many *steps* and *services* in parallel as possible.

![independent pipelines](/img/project_gantry/independent-pipelines.png)

Both pipelines *a* and *b* will execute their steps *0*, *1*, and *2* in order, but neither will *a1* wait for *b0* nor *b2* for *a1*.

The execution order is determined by topological sorting of the dependency graph.
Cycle detection is used to avoid unsatisfiable execution orders.
If a cycle is detected the execution is aborted.
This is required as it is not clear which step should be executed first.
Each *step* requests to be executed after another *step* and following the cyclical dependencies after itself.
\\(y\\) is dependant on \\(x\\) will be denoted as \\( x \rightarrow y\\).
Given $$ a \rightarrow b, b \rightarrow a$$ it is impossible to know if execution should start with either \\(a\\) or \\(b\\) as well as whether \\(a\\) or \\(b\\) should ignore the other to be able to start execution.

The need to ignore a dependency arises from the transitivity of preconditions.

\begin{align}
a \rightarrow b, b \rightarrow a
& \Rightarrow a \rightarrow b \rightarrow a, b \rightarrow a \rightarrow b \newline
& \Rightarrow a \rightarrow a, b \rightarrow b
\end{align}

Since \\(a\\) requires that \\(b\\) finishes successfully in order for it to be executed, and \\(b\\) requires the same from \\(a\\), \\(a\\) can only be executed after \\(a\\) finished successfully, the same holds for \\(b\\).
This implies that no step can be started before it finishes.

## gantry.env.yml

The `gantry.env.yml` file allows global configuration as well as meta-configuration of each defined *step* or *service*.

### Supported keywords

* `projectname`<br/>
   Overrides the calculated project name based on the folder containing the executed `gantry.yml`/`docker-compose.yml` file
* `substitutions`<br/>
  Key-Value mapping of values to expand variables (`${VAR}`).
  Can be accessed and altered by the preprocessor.
  Setting them in the calling environment can be used to replace the initially defined values as well providing them with `-e` flag, e.g. `-e FOO=bar`.
* `tempdir`, `string`, default: `/tmp`<br/>
  All temporary directories created using the preprocessor will use this as base.
* `tempdir_no_autoclean`, `bool`, default: `false`<br/>
  If set to `true` temporary directories created by the preprocessor will not be deleted by `gantry` and can be examined.
* `services` and `steps`<br/>
  List of meta-data for *services* and *steps*, follows the same structure as in `gantry.yml`/`docker-compose.yml`.

#### Supported keywords in *services* and *steps*

* `keep_alive`, default: `yes`<br/>
  Possible values are `yes`, `no`, and `replace`. `replace` and `yes` keep *services* running until the pipeline is run again.
  `yes` stops the *service* as soon as the pipeline execution starts.
  `replace` waits until the *service* is explicitly started.
  `no` stops the *service* as the pipeline finishes execution.
* `ignore`, `bool`, default: `false`<br/>
  If a *step* or *service* is marked as ignored, it will not be considered during planning or execution.
  It is also removed from any `depends_on` or `after` lists.<br />
  Using the `-i` command-line flag steps can be marked as ignored too.
  ([Example](https://github.com/ad-freiburg/gantry/tree/master/examples/partial_execution), [Example](https://github.com/ad-freiburg/gantry/tree/master/examples/selective_run))
* `ignore_failure`, `bool`, default `false`<br/>
  The execution of a pipeline is stopped as soon as a *step* exits with a non-zero exit code.
  Setting `ignore_failure: true` allows the pipeline to continue even if the *step* or *service* exits with a non-zero exit code.
  ([Example](https://github.com/ad-freiburg/gantry/tree/master/examples/diamond_ignore_failure))
* `exit_code_override`, `int`, default `0`<br/>
  If set to a non-zero value the exit code on failure will be replaced with the specified value.
  Non-zero exit codes propagate through gantry and will be returned to the calling shell.
  This can be used with scripts handling the exit code of `gantry` itself.
  ([Example](https://github.com/ad-freiburg/gantry/tree/master/examples/diamond_exit_code))
* `stdout`, `stderr`
  * `handler`, `string`, default: `<empty>`<br/>
    Allows to redirect the `stdout` or `stderr` of the *service* or *step*.
    The default prints to `stdout`/`stderr`.
    `file` will redirect the output into the `path`.
    `both` redirects and prints the output.
    `discard` prints nothing.
  * `path`, `string`, default:<br/>
    Optional for default handler and `discard`, required for `both` and `file`

#### Possible usage of `keep_alive`

The `keep_alive` flag allows replacing running services with minimal downtime.
Combining the following pipeline and `gantry.env.yml` enables the construction of a new search index for QLever and running basic tests on it, using the *new* service.

![QLever in-place rebuild.](/img/project_gantry/qlever-inplace-rebuild.png)

```
services:
  current:
    keep_alive: "replace"
  new:
    keep_alive: "no"
```

This keeps the *current* service running until the tests on *new* have finished successfully and the index has been moved to its destination to be used by *current*.
Only then *current* is stopped and immediately restarted with the new search index.
The final *test_current* warms the cache of the *current* instance, at the end of the pipeline the *new* instance will be shut down.

## Preprocessor

As mentioned in previous sections, gantry includes a preprocessor handling *variable expansion* (`${X}`), *preprocessor statements* (`#!`), and *comments* (`#`).
If an error occurs during preprocessing no *steps* or *services* will be started.

All preprocessor statements are lines prefixed by `#!`.
Whitespaces in front of `#!` are ignored as well as directly after `#!`.
This ensures that the following lines are processed by the preprocessor:
```
#! SET_IF_EMPTY ${A} X
    #! SET_IF_EMPTY ${B} Y
#!SET_IF_EMTPY ${C} Z
```

The preprocessor can be applied to a single file, printing the resulting file to `stdout` using `gantry preprocessor apply <file>`.

### Preprocessor statements
The list of currently implemented preprocessor statements can be viewed using `gantry preprocessor statements`.
At the time of writing this post the following statements are implemented:

* `CHECK_IF_DIR_EXISTS ${X}`, `check_if_dir_exists ${X}`<br />
  Aborts the execution if `X` does not hold the path of a valid directory.
* `SET_IF_EMTPTY ${X} Y`,`set_if_empty ${X} Y`<br />
  Sets variable `X` to `Y` iff `X` is empty or undefined.
* `TEMP_DIR_IF_EMPTY ${X}`,`temp_dir_if_empty ${X}`, `mktemp ${X}`<br/>
  If `X` is empty a temporary directory is created and the path stored in `X`.
  This uses the `tempdir` `gantry.env.yml` setting to determine the base and will be automatically deleted unless `tempdir_no_autoclean` is set to `true`.

## DOT graph

Using `gantry dot` the steps and services defined in the `docker-compose.yml`/`gantry.yml` will be stored as a directed graph in the `DOT` format.
The resulting `gantry.dot` file can be translated into images using the `dot` command-line tool from [Graphviz](https://graphviz.org/).
These images allow to see the explicit execution order as well as seeing how ignoring steps will influence the order and amount of executed *steps* and *services*.

All graph-images in the [examples](https://github.com/ad-freiburg/gantry/tree/master/examples) and this blog entry are created using both `dot` and `gantry dot`.

The direction of the arrows follows the order of execution, but can be reversed to point to dependencies using `--arrow-to-precondition`.

# Implementation

The execution of most (sub)commands can be divided into three parts:

1. Environment calculation
1. Pipeline generation
1. Pipeline execution

Some commands like `gantry dot` either skip or replace the *Pipeline execution* with own logic.

## Environment calculation

The environment for executing *steps* and *services* can be modified through different mechanisms.

Using `gantry.env.yml` defaults can be set, e.g. marking a *step* as ignored.
Substitutions defined in the file will be merged and replaced by the calling environment.
Additional command-line arguments are evaluated, which can change the environment (`-e`), mark additional steps as ignored (`-i`) and explicitly select subtrees (through explicitly naming containers to execute).

The execution is aborted if a *step* or *service* with conflicting values for *selected* to run and being *ignored* is encountered.

## Pipeline generation

To illustrate the pipeline generation a simple example with *two* independent pipelines will be used.

```
steps:
 a:
  ...
 b:
  ...
 c:
  ...
  after: a, b
 x:
  ...
 y:
  ...
  after: x
 z:
  ...
  after: y
```

The dependencies can be written as follows, where \\( \rightarrow \\) denotes the flow of execution.

$$ a \rightarrow c, b \rightarrow c $$
$$ x \rightarrow y, y \rightarrow z $$

The cycle detection and topological order are determined using a single run of a slightly modified implementation of [Tarjan's strongly connected components algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm).
As soon as a missing dependency is encountered the execution of Tarjan's algorithm is aborted and no dependency graph is generated.

If all dependencies are present, the graph is returned as a list of strong components.
These strong components are either a *single vertex* or a *cycle*.
If a *cycle* is found the execution stops listing all vertices contained in the *cycle*.
This cycle-detection simply checks that each strong component only consists of a *single vertex*.
The result must not be unique as the order of *a* and *b* does not change anything for the dependency graph, thus the following results are equal:

$$ (\\\{a\\\}, \\\{b\\\}, \\\{c\\\}, \\\{x\\\}, \\\{y\\\}, \\\{z\\\}) $$
$$ (\\\{b\\\}, \\\{a\\\}, \\\{c\\\}, \\\{x\\\}, \\\{y\\\}, \\\{z\\\}) $$

Starting at the right-most entry the components are then reordered into independent pipelines.
This is possible as the result of Tarjan's algorithm groups connected components into nearby lists.
Connectivity is evaluated by starting at the highest index and adding steps to the pipeline until all dependencies are fulfilled.
As soon as a non-connected vertex is encountered one pipeline is complete and a new pipeline is started.
The execution order inside a pipeline is achieved by prepending each encountered step to the current list of steps.
Thus resulting in one of the following: 

$$ \\\{(a, b, c), (x, y, z)\\\} $$
$$ \\\{(b, a, c), (x, y, z)\\\} $$

The order of the pipelines is not relevant as both \\((a, b, c)\\) and \\((x, y, z)\\) will be executed at the same time.
Only the order inside each pipeline is relevant for the creation of the explicit dependencies.

## Pipeline execution

`gantry` is written in [Go](https://golang.org/) and makes heavy use of [Goroutines](https://golang.org/doc/effective_go.html?#goroutines), [WaitGroups](https://golang.org/pkg/sync/#WaitGroup), and [Channels](https://golang.org/doc/effective_go.html?#channels) in the execution of the pipelines.

A single *WaitGroup* is defined to keep `gantry` alive and running until all steps have finished.
A single *abort* channel is defined, which is used to signal each container to skip execution after an error occurred.
A single *run* channel is defined and added to each step as a additional precondition.
This ensures that the execution is only started after all container and dependencies are explicitly instantiated.

For each container a channel is defined, which is added to all dependent containers as precondition.

Each container is then started as a *goroutine* which waits until all *preconditions* are satisfied, if the *abort* channel signals to abort, execution will be skipped.
This ensures that no container can run out-of-order and allows for multiple steps to be executed simultaneously.
After a step finishes it can signals on the *abort* channel, if the underlying container finishes with a non-zero exit-code.
If the step is instructed to ignore the failure through the `ignore_failure` flag, the error is ignored by the other containers.
Finally, the containers own channel is closed, allowing the dependents to progress their dependency checks.

After all containers are marked as finished, gantry can proceed to clean-up and exit.

# Source code, executables, and more
The source code, executables, and more examples can be found on [GitHub](https://github.com/ad-freiburg/gantry) and are licensed under the Apache License 2.0.
If you miss a feature or found a bug, feel free to open a pull request.

