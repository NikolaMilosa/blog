---
title: "Part I: Bazel?"
date: 2024-01-19T18:53:51+01:00
draft: true
tags: [ "bazel", "c", "tooling" ]
series: Learning Bazel
---

The code I used can be found on this [repository](https://github.com/NikolaMilosa/coreutils)

## Introduction

Finally it came time when I can no longer write simple things in .net and call it a day. To be honest I've completed all of the previous posts in a day and left them to cook for about 24h more since I wanted to get a good night sleep before uploading. I haven't still written here anything I do for job that I find interesting. Yes, .net is something we are keen on in the firm but at the end of the day I don't think I've managed to learn really anything. I've just seen and finally put to practice all the ideas I had in my head.

Now that time comes to an end as we embark on a new journey where I will tell you a small story of how we came to this [series](/series/learning-bazel). At [Dfinity](https://dfinity.org/) we like to think that open source is the way to go and we wanted to have a monorepo where we will put all our code. The repo can be found [here](https://github.com/dfinity/ic) for all of you interested to see how the code looks like. 

## Teams

I myself am a part of DRE team (Decentralized reliability engineers) or Dr. DRE as we like to call ourselves and we don't commit often to said repository. To be honest even though I am the youngest in the team I've commited the most (code wise), to the repository, others usually worked on some other parts of configurations and what not. We use rust (of course) for the code which is wrapped up together with some python for scripts and voilla you get a nice functioning thing. We even wrote a docker image which was a so-called *dev-container* where one has all his deps configured and basically is ready to go. Bare in mind that we have around 20 teams of 5-6 people actively working which means a lot of commits and a lot of new deps and a lot of new containers.

## The problems

The first problem came to the image size. The image grew as the project grew and there came some points where no one really knew if we needed all that stuff. But noone dared to touch anything since our whole CI was dependant of the image and really it was a huge fuss if something was touched there. 

The second problem was huge CI times. On such a repository its super hard to merge code. We have one protected branch and all of us are racing for reviews on PR's and there are some dedicated runners for which we are figthing as well. No, to be honest I don't remember exactly but our CI times were around couple of hours and you would get all your checks, lints, tests, binaries and what not built and ready. The real pain was when someone requests a coding style change of renaming a variable, a function or something that wasn't really realated to the functionality but was related to code understanding. Then, you would have to wait another N amount of hours and you were stuck. To add on top, we didn't have any automatic builds of any kind of OCI images and we couldn't easily test them, which basically meant that while you are waiting for your CI to finish you can go grab lunch, have a coffee, run, have a family, raise a kid, help them graduate... You get the point...

The third and final problem was that, if you didn't want to use said image you would have to setup your environment to match the one in dockerfile exactly. The changes in the setup were not communicated and you could catch them only when you are trying to see what actually broke and you need to see if you are missing anything...

## Demonofieing

*I know its not a word*

Slowly but surely we started branching off to our stories. There are some teams that are tied to this repo and I wish them all the luck but our team had a great opportunity to say goodbye to all this and do it our own way. At the moment of writing this the repo can be found on this [link](https://github.com/dfinity/dre) and we are making our own mono repo for tools and other things. In the mean time the whole org switched to `bazel` and we wanted to do it to. And so we did. 

This series will be related to solving some problems which we had which weren't so straight forward and which were not `googlable`. Funny enough I've never seen a tool so badly communicated in the community. You can't have any standard ways to learn about it. Its basically a see of random posts and their docs and some examples which are too simple sometimes and you're on your own.

## My goal

I wanted to dedicate this series to my way of learning bazel. My collegue set up our workspace and some CI and we all just did our bits and pieces of adding the stack that we own and we kind of made it work. What I wanted to do is setup everything myself and see how I would do it and find my way through. When the time comes I will tell a story of what problem acutally made me interested in the whole topic to begin with but that will come in some weeks, until then we have to setup everything up and to get the basics together.