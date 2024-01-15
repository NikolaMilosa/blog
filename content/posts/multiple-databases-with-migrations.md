---
title: "Multiple Databases With Migrations"
date: 2024-01-15T12:15:05+01:00
draft: true
tags: [ ".net", "ef", "tutorial" ]
---
I'm kind of having fun with these dotnet deep dives and I've had some ideas pop here and there so I want to keep the theme going. There were a lot of times where I needed to quickly test something and I didn't have the whole database set and the one standard migrations setup wasn't plausible. To further enhance my point, I think that with the way migrations are explained its hard to test new databases since migrations tie you to a database type. That got me interested in the topic of having multiple types of databases configured and use them at demand.

The whole idea is to have one [docker-compose](https://docs.docker.com/compose/) file which can spawn all the services and make use of environment variables to tell my API what setup to use. 

## What are .net migrations?

There is an [article](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli) by Microsoft themselves explaining what are the migrations but I wanted to make a few additions. Given the quick pace code develops and the whole [DevSecOps](https://www.redhat.com/en/topics/devops/what-is-devsecops) there is a need to make everything present in source control. I quite like seeing neat repositories where there is everything setup for you. You have your documentation in `docs`. You have your scripts in `scripts` and so on... This is just anoter thing that is missing. Microsoft finds it easier to maintain long running projects by modifying data schemas incrementally. With which I agree. 

If I had a commit `a` and it has some code in it. If i align my database with all migrations present in commit `a` I expect that my API will conform with database and it should work.

## Alternatives

To be quite honest I don't know many alternatives to entity framework that work the same way. There are a number of [ORMs](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) that manage database and code links but none of them work the same way EF does. I've seen projects which try to use EF and [Dapper](https://github.com/DapperLib/Dapper) trying to gain performace from its beautiful reads which they do. But as the time flies they fail to migrate Dapper queries to align with code changes meaning in a lot of bugs. The real question is:
> Can EF echosystem and other ORMs be combined together?

To that I still don't have the answer. I've used [NHibernate](https://nhibernate.info/) which is (you would guess it) inspired by java. It doesn't have any way to manage the schema and if the code changes one would have to track database schema in a different way. You could use [Database Projects](https://learn.microsoft.com/en-us/sql/ssdt/how-to-create-a-new-database-project?view=sql-server-ver16) to track those and it would be okay. The problem then arises that you are tied to Microsofts SQL Server since they don't support any other database (who would've thought! 🤔).

## The problem

Put simple: 
> One cannot *easily* have multiple database migrations generated in the same project. But he can do it with multiple projects!

So lets see how to solve that!
