---
title: "Part II: Bazelifying Dotnet"
date: 2024-01-25T18:30:58+01:00
tags: [ "bazel", "dotnet", "tooling" ]
series: Learning Bazel
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/background-worker-example)

This has to be one of the worst experiences I've had dealing with some code regarding dotnet. To be honest first time in forever I was stuck bashing my head thinking will I be able to do this and at the end, I am not sure I truly did it. Anyway I am here to discuss what I found and what I managed to learn.

## Setup

So to learn the tool I think one has to get a lot of hands on experience with a tool and the whole ecosystem. Bazel has a lot of subcommunities within itself. Which is kind of expected. You have a tool that is built in a way where each language community has to come and add their things. I feel like nobody will add rules for a language if nobody needs them. To some extent expected, although this time it feels like this tool will never mature.

What I wanted to do is take a project I tackled in one of the previous posts and bazelify it. I chose the one from [backgournd worker](/posts/background-services-dotnet) example and wrapped it around bazel. What that would give is hermetic builds on each machines and technically one wouldn't need anything. Just bazel and you rock... Right...

**NOTE**: The statement above wasn't right, don't listen to *you just need* statements

What you actually need is:
1. dotnet - the tool, you can use any version you want since you will use bazel from now onwards!
2. [paket](https://fsprojects.github.io/Paket/) - a tool used for managing dependancies in .net. Its like nuget gallery but you have your deps in a different file. Prior to this I haven't honestly played with it so I don't have an opinion. I see clear benefits of using it in some places but in 90% of the times I feel like nuget gallery does the job
3. podman/docker - at the end we will get an image and we want to be able to run it and easily deploy it wherever.

## WORKSPACE and MODULE.bazel

This part was pretty easy and it worked right off the bat. Basically you go to [their releases](https://github.com/bazelbuild/rules_dotnet/releases) and copy the contents of `MODULE.bazel` and `WORKSPACE` into corresponding places. After that you have to configure `paket`. On their site I had to follow just a [Get started](https://fsprojects.github.io/Paket/get-started.html) guide and I did what I needed to do. When you have your `paket.dependencies` and `paket.lock` you can move to the next step.

Now we need to linkup those two! For that, good folks on the repository wrote a tool called `paket2bazel` which can be found [here](https://github.com/bazelbuild/rules_dotnet/tree/master/tools/paket2bazel). Its nice to write the correct mappers for such tedious things since my current bazel experience has been just that. Tedious. Usually writing more things than I really should.

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

Now there is a couple of things on which I would like to comment. *First*: **THE ORDER IS IMPORTANT**. You cannot do 1. and 3. and then 2. since bazel will not be able to evaluate whole `WORKSPACE` hence you won't be able to run the command... That is tedious but okay, memorable... *Second* **IN THEIR DOCS THEY MISPELLED IT**. Probably in the speed or in the development they forgot to update the steps and for someone who just wants to add something that works you really lose a lot of time on such weird things. If I was better at bazel I would've noticed it right away but here I am ranting about README files. Who would've thought 😢.

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
├── bazel-BackgroundWorkerDotnet 
├── bazel-bin 
├── bazel-out 
├── bazel-testlogs 
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

Before going forwar I want to say what I mean by *bazelify*. I want to be able to do following things with my app:
1. I want to be able to build and run the app
2. I want to be able to build an OCI image which I can publish to some container registry
3. I want to be able to run image from step 3.

This sounds like a walk in the park 🌲!

### Building and running the app with bazel

With some better structuring I could've written `glob([])` for specifying what files should be included in the binary itself but since I had only a few files I've gone and manually entered them. The first step is to create your `BUILD.bazel` for your app. 

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

### Packaging and building an image

And this is where the pain and agony begin. This where you are pretty much stuck on your own. You have to read the docs multiple times. You can't really make anything out of hello world examples, even though you, yourself may be building a hello world example... But don't worry, I've lost some hair so you can do it easier.

Even though this may sound easy you really have to work for it. The first thing to do is to publish the binary locally. You can do that like this:
```python
publish_binary(
    name = "background-worker-publish",
    binary = ":background-worker",
    self_contained = True,
    target_framework = "net7.0",
    runtime_packs = [
        "@paket.main//microsoft.aspnetcore.app.runtime.linux-x64",
        "@paket.main//microsoft.netcore.app.runtime.linux-x64",
    ]
)
```
What this does is actually packages the app and really is equivalent to a `dotnet publish`. What is really cool is the `self_contained` part which adds some dependencies to your app so it doesn't depend of dotnet when you deploy it since it will be packaged within. Maybe its just me, maybe its all of us but I was stuck here since for me *publishing* is related to doing some kind of a *push* and I didn't even notice this rule for a long time. Now to collect all the things that are output from the publish we need to add another archive in `WORKSPACE.bazel`:
```python
http_archive(
    name = "rules_pkg",
    sha256 = "8f9ee2dc10c1ae514ee599a8b42ed99fa262b757058f65ad3c384289ff70c4b8",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_pkg/releases/download/0.9.1/rules_pkg-0.9.1.tar.gz",
        "https://github.com/bazelbuild/rules_pkg/releases/download/0.9.1/rules_pkg-0.9.1.tar.gz",
    ],
)

load("@rules_pkg//:deps.bzl", "rules_pkg_dependencies")

rules_pkg_dependencies()
```
Alongside a small change in `MODULE.bazel`:
```python
bazel_dep(name = "rules_pkg", version = "0.9.1")
```
With this new toy we are able to do some nasty packaging and bundling of things. This set of rules is really a story for it self. What we will use it for is to package everything as `.tar` and continue forward. To do that we have to modify our `BUILD.bazel` with following lines:
```python
pkg_tar(
    name = "background-worker_layer",
    srcs = [":background-worker-publish"],
    include_runfiles = True,
)
```
After adding that and the corresponding `load()` at the top of the file you should be able to run something like `bazel build //src:background-worker_layer` and you should get packaged tar on the route `bazel-out/k8-fastbuild/bin/src/background-worker_layer.tar`. I wouldn't even mention this if this wasn't super helpful to me for debugging. See, in bazel world you are usually on your own and having this is really great since you can inspect what is packaged and how in the tar file. Usually some paths will be off and with this you can look for correct paths.

Now for the part of building an OCI image. In recent times I wanted to frustrate myself with building *THE SMALLEST POSSIBLE* images and tried to go as small as possible and usually had to deal with a lot of pain from that. So I wanted to do the same with this one. Again lets add some more tools to the `WORKSPACE.bazel`:
```python
load("@rules_oci//oci:dependencies.bzl", "rules_oci_dependencies")

rules_oci_dependencies()

load("@rules_oci//oci:repositories.bzl", "LATEST_CRANE_VERSION", "oci_register_toolchains")

oci_register_toolchains(
    name = "oci",
    crane_version = LATEST_CRANE_VERSION,
)

load("@rules_oci//oci:pull.bzl", "oci_pull")
oci_pull(
    # tag = bookworm-20231218-slim
    name = "debian-slim",
    digest = "sha256:45287d89d96414e57c7705aa30cb8f9836ef30ae8897440dd8f06c4cff801eec",
    image = "index.docker.io/library/debian",
)
```
> I just wanted to take a minute and say how frustrating docker and dockerhub is. They had to do everything their own way and the pain of finding the correct url was immense. Anyway lets move on...
After this we can add a couple of more things to `BUILD.bazel` and get the following result:
```python
oci_image(
    name = "background-worker-image",
    base = "@debian-slim",
    entrypoint = ["/background-worker"],
    tars = [":background-worker_layer"],
    env = {
        "DOTNET_SYSTEM_GLOBALIZATION_INVARIANT": "1"
    }
)

oci_tarball(
    name = "background-worker-tarball",
    image = ":background-worker-image",
    repo_tags = [ "ghcr.io/nikolamilosa/background-worker-example/background-worker:latest" ]
)
```
With this we've completed the setup for the things we've set out to do. When you build the image you can import it in your local container registry with:
```bash
p load --input bazel-out/k8-fastbuild/bin/src/background-worker-tarball/tarball.tar
```

Please for the love of god save yourselves some pain and use [`dive`](https://github.com/wagoodman/dive). Once you build and import the image you will probably have some runtime errors which will need addressing. Be it paths or whatever you can inspect images with this tool and I use it daily.

I managed to build a whole image with 177MB which is great for an interpreted language that depends on a runtime.

## Conclusion

Generally is using paket better than microsofts package manager? Well I don't know. As far as I know its pretty hard to specify a certain sha256 of a package you are expecting. Imagine you want to use Microsoft.EntityFrameworkCore version 8.0.1. You would go and add it via its manager probably in Visual Studio or maybe through CLI with `dotnet add package Microsoft.EntityFrameworkCore --version 8.0.1`. What you would expect is that now everyone who is using that package gets the same library. With using paket you can have some benefits since you can get it to require certain commits or SHA of a package so you know that the package itself wasn't tampored with. It is more declarative and I think that its going to be more and more helpful since we are entering the era of full automatization and being 100% sure you know what is going in and out of the app fully is really necessary.

Having said that here is a direct counter to that argument with a next observation. Lets say your package requires a dependency A. Now that dependency has a dependency on a package B. You would **yourself** have to specify that both of those packages should be included in `csharp_binary`. This was a little frustrating but I believe that there has to be a way to write a simple script that generates what should be put in `csharp_binary` deps so you don't have to dig in and find out everything. 

What I've learned is how bad bazel is for languages that have a runtime. Maybe its just its ecosystem or maybe its just that it was thought out in a way for projects that are more thought through. It feels like you have to do a lot of heavy lifting to find out the direct meaning of the documentation itself. If I didn't have `dive` and manually inspected the layer I wouldn't be able to nail down the correct paths. And in the end I didn't even need those. All in all, I think bazel has its place since its caching mechanism is great and can be super benefitial to save time in testing and building, but I wouldn't recommend it to everyone and to everything.

There is still some playing I want to do with bazel and I have in mind some more ideas to extend this series which will hapen for sure. The journey of bazel is just beginning and I am excited to see what else is there to be found out. 

Until then, cheers!