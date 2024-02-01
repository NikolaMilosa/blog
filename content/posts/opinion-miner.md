---
title: "Opinion Miner"
date: 2024-02-01T14:30:58+01:00
draft: true
tags: [ "python", "llm", "rust" ]
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/opinion-miner/tree/10bbff3dda728469842d6a681d35150f9d0cf404).

This has to be the first time my head flooded with ideas on how can this be turnt into a large scale project. Even a micro-saas! Maybe this blogging thing is starting to produce its benefits! This weeks work was directed towards a POC for an opinion miner that I wanted to create. The idea is the following:
> When someone needs to find out what does the general public think of a certain topic who to ask?

Googleing is tedious as you need to spend hours reading blogs and the problem is that even if you do that a lot of content is AI generated and you really waste your time there. So how about automating it?

## The problem

I wanted to build a system that can be easily used and the whole point of it is to get all sorts of data and sentiment analysis about a certain topic. I see this extreamly fit for small to mid businesses that require data about the market and want to get a feel for a certain topic. The underlying topic of sentiment analysis is something that can be tackled in many ways and probably should be. Since the automation is used for something that is really *subjective* a vast amount of data should be provided that were gathered with different techniques so the user can decide wether to trust the data or not.
