---
title: "Long running jobs in ASP.NET"
date: 2024-01-18T12:58:31+01:00
draft: true
tags: [ "tutorial", ".net" ]
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/background-worker-example)

Right around the time when I started getting involved in our lovely sport of programming cloud was becoming a big thing. To understand the whole cloud one would have to spend a lot of time deep diving into certain topics and bits and pieces of software. Why cloud became super popular is because of startups. You could really quickly deploy your idea somewhere and test if it is working. Generally you wouldn't need to buy a server, configure networking, power supplys and many many other things which is hard to do right for one person.

On top of that there were these things called Azure Functions, AWS lambdas and whatever are they called in GCR. They serve a great way to spinup some kind of environment for a long running task so you don't overload your API with work. What cloud brought to the table was amazing. You could really achieve a lot and pay for what you use only. What it also produced is a real big gap in knowledge and we really are losing good software architects. To do a certain thing you can do it in multiple ways on cloud and probably achieve okay results. But it came at the expense of us not needing to learn the *whole-whole* stack of our systems. We don't need to learn about first couple of network layers because we really don't need those and those are handled. What we need is some code and thats it. Or is it?

My main problem with cloud is that its a business model rather than a solution. It is there so that cloud providers profit and not to help developers. They really do want our stacks to be bad, inefficient and faulty. They will do their best to help you with a more expensive tier and [pay-as-you-go subscriptions](https://www.rebilly.com/blog/pay-as-you-go-versus-subscription/). If we learnt our architecture and if we were to really dive deep in to the topic of observability, monitoring and provisioning we would quickly see that it was meant to break.

## Is it all bad?

Short answer *no*. 

If you want to test out your idea, business or service you could easily deploy it and see how it behaves. You can even catch bugs and debug your way through easily. Where cloud failes is when you scale it out. It becomes super frustrating to deal with a lot of services and apps deployed to different areas and you cannot really grasp the whole system from one place. Once your idea proves good and you want to scale it out you should probably invest in some way of on-premise solutions since they provision way more control.

Why I became interested in this topic and why I wanted to write about it is because I was asked to take a look at some code and see how to optimize its *cost*. The first time I was asked to do that and to be honest I was really not sure what to do. So I began the investigation.

## Findings

In a firm I work for most of the tech stacks are .net based which means that all the code *our* people write is probably a .net application. It will be deployed to azure most of the time and everything that is faulty will be fixed in .net. Is it nice? I don't know. I find .net nice and easy to work with but I don't like the way it is portraited as a silver bullet when its really good only for web development. Anyway, I found out that the project had a lot of azure functions that were also writen in .net and were really used just to map some things to another in the DB. Project uses the most expensive app plan on azure and they have about 50-60% of CPU free around 70% of the time. So my question was why didn't they migrate their functions to background services?