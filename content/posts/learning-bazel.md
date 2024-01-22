---
title: "Part I: Bazel?"
date: 2024-01-19T18:53:51+01:00
tags: [ "bazel", "c", "tooling" ]
series: Learning Bazel
---

## Introduction

Finally it came time when I can no longer write simple things in .net and call it a day. To be honest I've completed all of the previous posts in a day and left them to cook for about 24h more since I wanted to get a good night sleep before uploading. I haven't still written here anything I do for job that I find interesting. Yes, .net is something we are keen on in the firm but at the end of the day I don't think I've managed to really learn anything. I've just seen and finally put to practice all the ideas I had in my head.

Now that time comes to an end as we embark on a new journey where I will tell you a small story of how we came to this [series](/series/learning-bazel). At [Dfinity](https://dfinity.org/) we like to think that open source is the way to go and we wanted to have a monorepo where we will put all our code. The repo can be found [here](https://github.com/dfinity/ic) for all of you interested to see how the code looks like. 

## Teams

I myself am a part of DRE team (Decentralized reliability engineers) or Dr. DRE as we like to call ourselves and we don't commit often to said repository. To be honest even though I am the youngest in the team I've commited the most (code wise) to the repository, others usually worked on some other parts of configurations and what not. We use rust (of course) for the code which is wrapped up together with some python for scripts and *voilla!* you get a nice functioning thing. We even wrote a docker image which was a so-called *dev-container* where one has all his deps configured and basically is ready to go. Bare in mind that we have around 20 teams of 5-6 people actively working which means a lot of commits and a lot of new deps and a lot of new containers.

## The problems

The first problem came to the image size. The image grew as the project grew and there came some points where no one really knew if we needed all that stuff. But noone dared to touch anything since our whole CI was dependant of the image and really it was a huge fuss if something was touched there. 

The second problem was huge CI times. On such a repository its super hard to merge code. We have one protected branch and all of us are racing for reviews on PR's and there are some dedicated runners for which we are figthing as well. Now, to be honest I don't remember exactly but our CI times were around couple of hours and you would get all your checks, lints, tests, binaries and what not built and ready. The real pain was when someone requests a coding style change of renaming a variable, a function or something that wasn't really realated to the functionality but was related to code understanding. Then, you would have to wait another N amount of hours and you were stuck. To add on top, we didn't have any automatic builds of any kind of OCI images and we couldn't easily test them, which basically meant that while you are waiting for your CI to finish you can go grab lunch, have a coffee, run, have a family, raise a kid, help them graduate... You get the point...

The third and final problem was that, if you didn't want to use said image you would have to setup your environment to match the one in dockerfile exactly. The changes in the setup were not communicated and you could catch them only when you are trying to see what actually broke and you need to see if you are missing anything...

## Demonofieing

*I know its not a word*

Slowly but surely we started branching off to our stories. There are some teams that are tied to this repo and I wish them all the luck but our team had a great opportunity to say goodbye to all this and do it our own way. At the moment of writing this, the repo can be found on this [link](https://github.com/dfinity/dre) and we are making our own mono repo for tools and other things. In the mean time the whole org switched to `bazel` and we wanted to do it to. And so we did.

This series will be related to solving some problems which we had which weren't so straight forward and which were not *googlable*. Funny enough I've never seen a tool so badly communicated in the community. You can't have any standard ways to learn about it. Its basically a sea of random posts and their docs and some examples which are too simple sometimes and you're on your own.

## My goal

I wanted to dedicate this series to my way of learning bazel. My collegue set up our workspace and some CI and we all just did our bits and pieces of adding the stack that we own and we kind of made it work. What I wanted to do is setup everything myself and see how I would do it and find my way through. When the time comes I will tell a story of what problem acutally made me interested in the whole topic to begin with but that will come in some weeks, until then we have to setup everything up and to get the basics together.

## The build concepts

This post will be more of a reading journey and a discussion of their [docs](https://bazel.build/start). I would like to deep dive into some action right away but I feel like me and bazel started of on a wrong foot you know... I haven't really read any documentation, any concepts, nothing. It was just plain head bashing and random posts here and there until I didn't start to get interested in the topic. 

So to do it right I want to dedicate a post to the concepts. They themselves state that there are six concepts:
1. [Workspaces, packages & targets](#1-workspaces-packages--targets)
2. [Labels](#2-labels)
3. [`BUILD` files](#3-build-files)
4. [Dependencies](#4-dependencies)
5. [Visiblity](#5-visiblity)
6. [Hermeticity](#6-hermeticity)

And that is the order in which I will cover them so here goes!

### 1. Workspaces, packages & targets

> Bazel builds software from source code organized in directory trees called repositories. A defined set of repositories comprises the workspace. Source files in repositories are organized in a nested hierarchy of packages, where each package is a directory that contains a set of related source files and one BUILD file. The BUILD file specifies what software outputs can be built from the source.

So to have a workspace one needs to create a directory with `WORKSPACE.bazel` file. For now the contents of the file can be as follows:
```python
workspace(name = "playground")
``` 
The language they use is called [starlark](https://bazel.build/rules/language) to which I will probably dedicate a whole post later on in this series. Right away if you run `bazel build` it will do a couple of things:
1. It will install the newest version of the tool
2. It will add a couple of more directories and folders which the tool uses
3. Download some deps it needs

So lets create a folder structure that resembles the following
```bash
.
├── BUILD.bazel
├── MODULE.bazel
├── MODULE.bazel.lock
├── more-repos
│   └── source
│       ├── BUILD.bazel
│       └── script.sh
├── repos
│   ├── BUILD.bazel
│   └── source
│       ├── BUILD.bazel
│       └── script.sh
└── WORKSPACE.bazel
```
What I wanted to test is if the `BUILD.bazel` is required in all the directories for the target to be discoverable but its not. If you run the following command you can see all the targets in your workspace:
```bash
bazel query ...
//more-repos/source:script
//repos/source:script
```
For the sake of completness (if someone is following along the boring part) here is the content of both `more-repos/sources/BUILD.bazel` and `repos/sources/BUILD.bazel`:
```python
sh_binary(
    name = "script",
    srcs = ["script.sh"]
)
```
If you want to build a target you can run `bazel build //repos/sources:script` or if you want to run you can do `bazel run //repos/sources:script`. Pretty neat stuff so far.

There is a handy definition by the bazel team which if I read earlier I wouldn't need to test the hypothesis but no pain no gain:
> A package is defined as a directory containing a BUILD file named either BUILD or BUILD.bazel. A package includes all files in its directory, plus all subdirectories beneath it, except those which themselves contain a BUILD file. From this definition, no file or directory may be a part of two different packages.

To clarify they've further added explainations to what are files in their system. A file can be either *source* file which are written by people, or a *generated* file (sometimes called *derived* or *output* files) which are generated from *source* files. Files are the first kind of **targets**

The second type of **targets** are declared with a *rule*. A *rule* is basically a relationship between a set of input and a set of output files. The inputs to a rule may be source files or the output of other *rules*.

### 2. Labels
There is a lot of things here and one cannot fully understand the meaning of the concept until we do a real dive into practices. What I will leave here is a quote from them since I've already experienced this and I haven't even noticed:
> A common mistake in BUILD files is using //my/app to refer to a package, or to all targets in a package--it does not. Remember, it is equivalent to //my/app:app, so it names the app target in the my/app package of the current repository.

### 3. `BUILD` files
So put easily BUILD files declare *targets* by invoking *rules*. To explain this lets look at the example from the top again:
```python
sh_binary(
    name = "script",
    srcs = ["script.sh"]
)
```
Here we see a declaration of a `script` **target** using the `sh_binary` rule. Every rule has a name attribute that declares a target within the package of the BUILD file. Since each rule is like a `function` it has a set of attributes that are applicable for a given rule which can be passed in the rule. In the example above we see that we have an attribute called `srcs` which is a common attribute for a lot of rules.

One important note from the Bazel team:
> The BUILD file can be named either BUILD or BUILD.bazel. If both files exist, BUILD.bazel takes precedence over BUILD. For simplicity's sake, the documentation refers to these files simply as BUILD files.

One thing I didn't know but I followed anyway is that BUILD files cannot contain function definitions, `for` statements or `if` statements which need to be defined in `.bzl` files instead. Also `*args` and `**kwargs` arguments are not allowed in BUILD files.

### 4. Dependencies
I think there is no need tu explain what are dependencies but I will spend a couple of words in writing what types of deps does Bazel know. They state that there are three types:
1. `srcs` deps - Files consumed directly by the rule or rules that output source files
2. `deps` deps - Rule pointing to separately-compiled modules providing header files, symbols, libraries, data, etc.
3. `data` deps - Some build files to run correctly. These files aren't source code. Good example of this may be some file used for unit testing or something similar.

One thing noted here is that we can use `glob(["testdata/*"])` to reference a directory and not have to go over targets one by one, which is pretty neat.

### 5. Visiblity
Right of the bat we have two types of visibility which we will cover:
1. Target visibility
2. Load visibility

#### Target visibility
Target visiblity controls who may depend on your target - that is who may use your target's label inside an attribute such as `deps`. What I find nice about this whole text is that they give us a *best practice* that states:
> To make several targets visible to the same set of packages, use a package_group instead of repeating the list in each target's visibility attribute. This increases readability and prevents the lists from getting out of sync.

To quickly write down all visiblity possibilities we have:
1. `//visiblity:public` - grants access to all packages
2. `//visibility:private` - does not grant any additional access; only targets in this package can use this target.
3. `//foo/bar:__pkg__` - grants access to `//foo/bar` and all of its direct and indirect subpackages.
4. `//some_pkg:my_package_group` - grants access to all of the packages that are part of the given `package_group`.

#### Load visibility
Load visibility controls whether a `.bzl` file may be loaded from other BUILD or `.bzl` files outside the current package. There is more things to be said here but I think its better to learn this part by example. Modeling any kind of code is hard at first and as one gets better he can observe how to structure things correctly, so this whole chapter is just a pivoting stone for the future.

### 6. Hermeticity
The idea is that when given the same input source code and product configuration, a hermetic build system always returns the same output by isloating the build from changes to the host system. In order to isolate the build, hermetic builds are insensitive to libraries and other software installed on the local or remote host machine. They depend on specific versions of build tools, such as compilers and dependencies, such as libraries. This makes the build process slef-contained as it doesn't rely on services external to the build environment.

## Conclusion 
The stated benefits are:
1. speed - since the output can be cached
2. parallel execution - with the calculation of [DAG's](https://en.wikipedia.org/wiki/Directed_acyclic_graph) one can calculate how to optimize for parallel execution
3. multiple builds - possiblity of multiple hermetic builds on the same host
4. reproducibility - hermetic builds are good for troubleshooting because you know the exact conditions that produced the build

Now all of this is nice and it sounds as exciting. In the following posts I will see what is my opinion on the system and if I like it or not. Generally if this is configured correctly it sounds really powerful and I am excited to see the whole story!

Until then, cheers!
