---
title: "Hugo Setup"
date: 2024-01-09T19:21:53+01:00
tags: [ "tutorial", "go", "cloudflare" ]
---

I guess the most logical way to start is to tell a story about this blog. I was never a frontend guy myself and probably never will be although I hope that this journey might at least give me more insights into it. I knew I didn't want to code any frontend and that I just want to create a blog, write stuff and get it done with. Maybe I looked at it from a wrong perspective since I really wanted to have something deployed and, well, presentable.

## Research

Looking at the current situation there are many options on how one can lead his blog. A lot of those options boild down to one thing and that is *Website builders & Static Site Generators*. What I wanted for this blog is to be versioned (possibly with git and on github) and easy to setup. What I've seen comes down to the following options:
1. Build your own
2. WordPress
3. Mkdocs
4. Others
5. Hugo (spoiler, this is what I chose)

## Building my own static site generator

I was never a fan of writing everything from scratch. I agree that one should probably write some code from 0 in order to learn, but that is not something I would ever recommend for production. I was just a couple of days ago reading an article that is titled [*We fired our top talent. Best decision we ever made*](https://www.freecodecamp.org/news/we-fired-our-top-talent-best-decision-we-ever-made-4c0a99728fde/) and the whole topic to me was interesting but one of the key points is that maintaining hand made tools can be hard espesially if they are written by small subset of individuals in the firm which can severly slowdown development. If your goal is not to learn about static site generators, which I am not saying you shouldn't just stating that I didn't really care about (at the moment), I would advise you not to go with this option.

## [WordPress](https://wordpress.com)

Even though a good option I anticipate that I will never be interested in php. I feel WordPress has its use with people that are not super into software development and just want some kind of double click programming, drag and drop, copy and paste and *voila*! Something that works is there. General feal is that if you don't care about tech and just want a general purpose blog and your mom can take over if you don't have any ideas I strongly suggest using WordPress. On this [site](https://wordpress.com/create-blog/) you can do everything in 5 minutes and it will just work. They even have a free tier, which is good for some small sites, but the suggested tier for blogs is \\$8 a month or \\$96 a year if billed anually which is acceptable if you don't wanna tweak a lot and experiment. If expert tier is taken you have live chats handled with some okay analytics. Since I don't want any profit from this blog and I only care about creating content I didn't go with this option.

## [Mkdocs](https://mkdocs.org/)

Now we are starting to dive into the things I really wanted. This option is a really popular way to host documentation, especially on github. The biggest selling point is that you can even get simple free hosting with github pages, but to be honest you can achieve the same with different generators its just that they didn't advertise it in the same way. I've had some experience with jekyll for an internal project in our company and I've set it up for documentation. The result is cool, you can see it [here](https://sotex-lab.github.io/staff-hub/) (if the repo is still in place). It supports a lot of themes and it wasn't hard to add them or tweak them. Its biggest con is that it is written in python and I really hate dealing with python in anyway. 

The best way to handle python in my opinion is to have some kind of pyenv setup, and then use poetry for different projects you want to run. Each theme is a pip dependency that has to be managed and that is the only thing I really didn't like. It is highly customizable and I think it suits some needs but in the end I wanted to try something new.

## Others

When deciding on how to setup this blog I was reading this post on [medium](https://medium.com/@ezinneanne/top-ten-popular-static-site-generators-ssg-in-2023-e1894fca6925) which lists some options and compares them. All in all a great piece if you are really interested in the topic.

## [Hugo](https://gohugo.io/)

In comes hugo! Loved the guy when I was young. Now, a whole 12-15 years later here I am using it for building a blog. Life can be a lot of things! Hugo claims to be the fastest tool of its kind which is expected since it was built in golang. My knowledge of go is still limited but that is something I will explore in the future. As of now, when I read:
> Hugo is the fastest tool of its kind. At <1 ms per page, the average site builds in less than a second.

I knew I had to use it just so I really see the limits of it. To be honest it was recommended to me by a colegue and after playing with it for a day I said I am going with it.

### Creating a blog
What I liked is the speed I had while setting up the hello world part of it. After installing go and hugo I was ready to *go* (pun intended).
~~~bash
hugo new site blog
cd blog
git init
git submodule add <repo-of-theme> themes/<name>
echo "theme = '<name>'" >> hugo.toml
hugo server
~~~

### Setting up

I used theme called [poison](https://themes.gohugo.io/themes/poison/) because it doesn't have unnecessary js and it is super fast. Having set that up now comes the part of setting your `config.toml`. To be honest I've just yanked everything from poison themes config, removed what I don't need and this is the result. The only real issue I had is that it wasn't stated *clearly* where to put the images so they load but after hard 3 minutes of head scratching I found out they have to go in `static` folder.

My setup looks like this right now, It may change in the future so don't hold me on this:
~~~toml
baseURL = "/"
title = "Mili writes tech"
author = "Nikola Milosavljevic"
copyright = "Nikola Milosavljevic"
paginate = 10
languageCode = "en"
theme = "poison"
pluralizelisttitles = false

[params]
    brand = "Mili writes tech"
    brand_image = "/images/image0.jpeg"
    description = "Exploring Code Horizons: A Young Developer's Journey"
    favicon = "/images/image0.jpeg"
    front_page_content = [ "posts" ]

    menu = [
        {Name = "About", URL = "/about/", hasChildren = false},
        {Name = "Posts", URL = "/posts/", Pre = "Recent"},
    ]
    email_url = "mailto://nikolamilosa20@gmail.com"
    github_url = "https://github.com/nikolamilosa"
    linkedin_url = "https://linkedin.com/in/nikolamilosa20"
[taxonomies]
    tags = 'tags'
~~~

I am a simple guy, I like things clean and easy and this is exactly it for me.

### Cloudflare

I've opted for cloudflare as a hosting platform and a domain registration service. Basically when you register and login [here](https://dash.cloudflare.com/) you need two things. First thing is the domain and you can register one on the *Domain Registration* part of the left sidebar. When you pick a name, you pay and its yours. Be mindful of wether you want to make it a recuring payment since it can cost a lot. After that you should pick *Workers & Pages* and press on *Create application* button. On top you will be able to choose between *Workers* and *Pages* for which you can choose pages and connect your Github profile. Onces connected you can pick a repository and walk through the UI to complete the setup.

When your site is up you will get a url for it, something along the lines of https://blog-au8.pages.dev (this is for mine but I've setup a c-name for it so it routes to the original domain). You want to go and change the *Custom Domains* and add your bought domain from earlier.

Last thing you need to do that I struggled with is to go to your domain in the cloudflare dashboard (click on the domain) and select *DNS* from the left sidebar. If the *Proxy status* is on you should disable it so that it shows *DNS only* which will then activate your domain and you will be able to visit the site.

## Conclusion

This is how I did it. It isn't perfect and it isn't a silver bullet. Probably it isn't even fully right. But hey, I've achieved my mini life goal which is leading a blog so I can take it of my bucket list! I am yet to setup any way of commenting so for now I guess you can email your opinion or maybe even write it as a discussion point on the repository of this [repo](https://github.com/NikolaMilosa/blog/discussions). 

Until next time, see you!