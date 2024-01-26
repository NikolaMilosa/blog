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

Now part that wasn't correctly discussed is the setup that is needed to load paket deps fully. Thankfully when you follow examples from the `examples` folder of their repository you can find everything you need. What is missing is to add to `MODULE.bazel` something like this:
```python
main_extension = use_extension("//:paket.main_extension.bzl", "main_extension")
use_repo(main_extension, "paket.main")
```

As for the essentials that is all. If you wanted to add a dependency you would have to go through the motion of running couple of commands:
1. Either add manually a dependency to `paket.dependencies` or run a `paket add <dependency>`
- Note that if you manually add a dependency you should run `paket install` just to regenrate the lock file.
2. Regenerate the `paket.main.bzl` and `paket.main_extension.bzl` files which you can do with
```bash
bazel run @rules_dotnet//tools/paket2bazel:paket2bazel.exe -- --dependencies-file $(pwd)/paket.dependencies  --output-folder $(pwd)
```

## Development with BAZEL

Since the solution previously was in top level of repository I had to refactor it. Previously it looked like this:
```bash
.
├── appsettings.Development.json
├── appsettings.json
├── BackgroundGenerator.cs
├── BackgroundWorkerDotnet.csproj
├── BackgroundWorkerDotnet.http
├── BackgroundWorkerDotnet.sln
├── Program.cs
├── Properties
│   └── launchSettings.json
```
When you add all your bazel files the issue you will run into is pretty common. In development of .net the team decided to switch from having to *include* all files that belong to a project explicitly to a model where you *exclude* files that don't belong to the project. Here you have two options:
1. Exclude all bazel related folders (namely `bazel-<sln>`, `bazel-bin`, `bazel-out`, `bazel-testlogs`) and continue working in the same toplevel directory.
2. Restructure the folder.

I went with the second approach. What I did is migrate to the following structure:
```bash
.
├── bazel-BackgroundWorkerDotnet -> /home/nikola/.cache/bazel/_bazel_nikola/19a8b2ab786c133f994eaf51038f4bfe/execroot/_main
├── bazel-bin -> /home/nikola/.cache/bazel/_bazel_nikola/19a8b2ab786c133f994eaf51038f4bfe/execroot/_main/bazel-out/k8-fastbuild-ST-bd42dc04ad95/bin
├── bazel-out -> /home/nikola/.cache/bazel/_bazel_nikola/19a8b2ab786c133f994eaf51038f4bfe/execroot/_main/bazel-out
├── bazel-testlogs -> /home/nikola/.cache/bazel/_bazel_nikola/19a8b2ab786c133f994eaf51038f4bfe/execroot/_main/bazel-out/k8-fastbuild-ST-bd42dc04ad95/testlogs
├── BUILD.bazel
├── MODULE.bazel
├── paket.dependencies
├── paket-files
│   └── paket.restore.cached
├── paket.lock
├── paket.main.bzl
├── paket.main_extension.bzl
├── src
│   ├── BackgroundGenerator.cs
│   ├── BackgroundWorkerDotnet.csproj
│   ├── BackgroundWorkerDotnet.sln
│   ├── bin
│   ├── BUILD.bazel
│   ├── obj
│   ├── Program.cs
│   └── Properties
└── WORKSPACE
```
And I quite like this. This makes you structure repositories correctly which I think is a whole different topic. I like having metafiles in the toplevel directory. What I would change personally is return the `BackgroundWorkerDotnet.sln` back to the toplevel directory and that would be a completed story in my opinion.

Having done this you are left to *bazelify* the app!

## Bazelifying ASP.NET core

This has to be one of more tedious jobs I've done in recent time. With some better structuring I could've written `glob([])` for specifying what files should be included in the binary itself but since I had only a few files I've gone and manually entered them. The first step is to create your `BUILD.bazel` for your app. 

When you declare a package you can add `csharp_binary` in there and mine looked like this:
```python
csharp_binary(
    name = "background-worker",
    srcs = [
        "BackgroundGenerator.cs",
        "Program.cs"
    ],
    project_sdk = "web",
    target_frameworks = [ "net7.0" ],
    targeting_packs = [
        "@paket.main//microsoft.aspnetcore.app.ref",
        "@paket.main//microsoft.netcore.app.ref",
    ],
    deps = [
        "@paket.main//swashbuckle.aspnetcore.swagger",
        "@paket.main//swashbuckle.aspnetcore.swaggergen",
        "@paket.main//swashbuckle.aspnetcore.swaggerui",
        "@paket.main//swashbuckle.aspnetcore",
        "@paket.main//microsoft.aspnetcore.openapi",
        "@paket.main//microsoft.openapi",
    ]
)
```
There is a few things to notice about this. First thing you have to be really *declarative*. That is the whole story with bazel, it assumes nothing, you have to basically learn it to walk each time and then it will walk the same all the time with you. It has some pros and cons which will be discussed later.

To explain what does this do. 
> This bazel *rule* adds a *target* named *background-worker* which can be *built* and *run*. It has two *source* files and uses a web sdk in addition to using net7.0 framwork. it is targeting aspnetcore and netcore packs built by microsoft which are present on nuget gallery. On top of that it has some dependencies explicitly stated in `deps` section.

One can notice that I switched from .net 8.0 back to .net 7.0. That happened since in the recent update microsoft added interceptors which aren't stabilized and are an experimental feature. `rules_dotnet` still didn't add any support for that and bazel cannot compile it for that reason. I couldn't find a workaround for that yet but our features allowed us to easily transition back one version and we are fine. 

Once all this is entered you can build it with: 
```bash
bazel build //src:background-worker
```
And you can run it with:
```bash
bazel run //src:background-worker
```

I've also made a small change in code in `Program.cs`
```diff
...
-app.Run();
+app.Run("http://localhost:8000");
...
```
Which tells the code on which port and host it should listen. I did it so I don't have to include `appsetings` and `launchProperties` to the mix since I can declare everything programatically.

## Packaging and building an image

## Conclusion

!TODO: talk about using nuget manager in parallel with paket or using paket only or anything else?
!TODO: talk about a bad dev exp since you have to manually add all deps in `csharp_binary`
!TODO: Maybe extension points