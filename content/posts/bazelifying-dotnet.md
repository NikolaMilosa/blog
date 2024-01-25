---
title: "Part II: Bazelifying Dotnet"
date: 2024-01-25T18:30:58+01:00
draft: true
tags: [ "bazel", "dotnet", "tooling" ]
series: Learning Bazel
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/background-worker-example)

This has to be one of the worst experiences I've had dealing with some code regarding dotnet. To be honest first time in forever I was stuck bashing my head thinking will I be able to do this and to be honest, I am not sure I truly did it. Anyway I am here to discuss what I found and what I managed to learn.

## Setup

So to learn the tool I think one has to get a lot of hands on experience with a tool and the whole ecosystem. Bazel has a lot of subcommunities within itself. Which is kind of expected. You have a tool that is built in a way where each language community has to come and add their things. I feel like nobody will add rules for a language if nobody needs them. Which is kind of expected. 

What I wanted to do is take a project I tackled in one of the previous posts and bazelify it. I chose the one from [backgournd worker](/posts/background-services-dotnet) example and wrap it around bazel. What that would give is hermetic builds on each machines and technically one wouldn't need anything. Just bazel and you rock... Right...

**NOTE**: The statement above wasn't right, don't listen to *you just need* statements

What you actually need is:
1. dotnet - the tool, you can use any version you want since you will use bazel from now onwards!
2. [paket](https://fsprojects.github.io/Paket/) - a tool used for managing dependancies in .net. Its like nuget gallery but you have your deps in a different file. Prior to this I haven't honestly played with it so I don't have an opinion. I see clear benefits of using it in some places but in 90% of the times I feel like nuget gallery does the job

To clarify you need dotnet just to install the paket tool and run it. 

## WORKSPACE and MODULE.bazel

This part was pretty easy and it worked right off the bat. Basically you go to [their releases](https://github.com/bazelbuild/rules_dotnet/releases) and copy the contents of `MODULE.bazel` and `WORKSPACE` into corresponding places. After that you have to configure `paket`. On their site I had to follow just a [Get started](https://fsprojects.github.io/Paket/get-started.html) guide and I did what I needed to do. When you have your `paket.dependencies` and `paket.lock` you can move to the next step.

Now we need to linkup those two! For that good folks on the repository wrote a tool called `paket2bazel` which can be found [here](https://github.com/bazelbuild/rules_dotnet/tree/master/tools/paket2bazel). Its nice to write the correct mappers for such tedious things since my current bazel experience has been just that. Tedious. Usually writing more things than I really should.

Anyway. A TL;DR is the following:
1. add the following to the `WORKSPACE`:
```python
load("@rules_dotnet//dotnet:paket2bazel_dependencies.bzl", "paket2bazel_dependencies")

paket2bazel_dependencies()
```
2. run the following command:
```bash
bazel run @rules_dotnet//tools/paket2bazel:paket2bazel.exe -- --dependencies-file $(pwd)/paket.dependencies  --output-folder $(pwd)
```
3. Add the following to your `WORKSPACE`:
```python
load("//:paket.main.bzl", "main")
main()
```

Now there is a couple of things on which I would like to comment. *First*: **THE ORDER IS IMPORTANT**. You cannot do 1. and 3. and then 2. since bazel will not be able to evaluate whole `WORKSPACE` hence you won't be able to run the command... That is tedious but okay, memorable... *Second* **IN THEIR DOCS THEY MISPELLED IT**. Probably in the speed or in the development they forgot to update the steps and for someone who just wants to add something that works you really lose a lot of time on such weird things. If I was better at the tool I would've noticed it right away but here I am ranting about README files. Who would've thought 😢.

TODO: speak about the adition to `MODULE.bazel` which wasn't documented as well.