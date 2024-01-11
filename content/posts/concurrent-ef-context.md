---
title: "Using .NET EF Context concurrently"
date: 2024-01-11T19:03:20+01:00
draft: true
tags: [ ".net", "ef", "tutorial" ]
---

Code for all the examples can be found on [this repository](https://github.com/NikolaMilosa/parallel-ef-context-example)

So a couple of days ago I was asked if I could take a look at the code that a collegue wrote. If I was to be honest about what I really like about my work that would be looking at unexpected issues people encounter and trying to solve them. I've never felt the same about discovering solutions to my problems and to problems of others probably because I usually expect my problems down the line whilst with others they just pop up and go randomly. 

Anyway a collegue has an ASP.NET Core API that is used for a billing project and one of their requirements is to create some sort of files that are reports of people doings every so often (Usually based on a request from some kind of frontend) and then return them a cheerful *OK!* once the process is done. Now they've noticed that in while processing an array of reports, each report is individual and is not dependant of any other reports. 💡 An idea popped!

> "Hey! This functionallity is really slow I was hoping we could optimize it somehow..."

One collegue said.

> "Could we make it run in parallel?"

And they embarked on the journey of finding out how their code cannot run in parallel since the EF context cannot be shared *safely* [between threads](https://stackoverflow.com/questions/54024103/share-dbcontext-among-threads). Now in my blog I vow to put close to 0 stack overflow references because I generally think that people are only flexing on that platform and that its just a bunch of younger devs who really don't have anything smarter to do. On top of that, you can find better results reading the docs but hey, here I am sitting and talking trash about something I've just done.

I know that probably what they could've done is refactor the code and parallelize later but what really got me interested is how *can* this be done and achieve safety even if it shouldn't be done like that. For science sake!

## Explaining the use-case

Lets say that I have an API that has an endpoint `/process` that when hit with `POST` method and given a body which looks like this:
~~~JSON
[ "x", "y", "z" ]
~~~
has to add them to the table called `Letters` where each object looks like this:
~~~c#
public class Letter 
{
    public int Id { get; set; }
    public char Char { get; set; }
}
~~~
and for debugging purposes after each addition it has to fetch all the letters and print a log of them.

## Sequential approach

The whole thing can be done in sequence and with little to no code. Idea is simple, I need to add some data then I need to save it so I can fetch and log everything. The whole code for this controller looks like this: 
~~~c#
[HttpPost("/sequentional")]
public async Task<IActionResult> Sequentional(char[] letters)
{
    var counter = 0;
    letters.Select(async x =>
    {
        await _appDbContext.AddAsync(new Letter
        {
            Char = x
        });

        await _appDbContext.SaveChangesAsync();
        _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", counter++, string.Join(", ", _appDbContext.Letters.Select(x => x.Char))));
    }).ToList();

    return Ok("Done");
}
~~~
The logs that we've got after this execution look like the following:
~~~bash
[Thread 0]: Current elements -> a, b, c, d, e, f, x
[Thread 1]: Current elements -> a, b, c, d, e, f, x, y
[Thread 2]: Current elements -> a, b, c, d, e, f, x, y, z 
~~~ 
Pretty boring stuff since there is nothing new with this. Now lets play with some ways to process something like this concurrently.

## The problem we face

To demonstrate what would happen if something like this would be naively implemented lets write an example for that. The code would look something like this:
~~~c#
[HttpPost("/bad-parallel")]
public async Task<IActionResult> BadParallel(char[] letters)
{
    try
    {
        await Parallel.ForEachAsync(letters, async (letter, cancellationToken) =>
        {
            var threadId = Guid.NewGuid();

            await _appDbContext.AddAsync(new Letter
            {
                Char = letter
            }, cancellationToken);

            await _appDbContext.SaveChangesAsync(cancellationToken);
            _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", threadId, string.Join(", ", _appDbContext.Letters.Select(x => x.Char))));
        });
    } catch (Exception e)
    {
        _logger.LogError(e.ToString());
    }
    return Ok("Done");
}
~~~
Now from a higher look one would expect that database handles concurrent requests and that you should be able to hit it concurrently without any issues. That is completly correct! Thank god for the database developers who knew that we would need something like that! The issue is in the `DbContext`, or the way it works in the background. See, when registering it in the DI container you would normally have some code that looks like the following: 
~~~c#
builder.Services.AddDbContextPool<AppDbContext>(x =>
{
    x.UseSqlite(AppDbContextFactory.ConnectionString);
    
});
~~~
Here we add a pool of context's (namely 1024 by default) which are instantiated when the first request hits the API. The pool manager then caches those connections and reuses them meaning that you don't have to open new connections to the database on each request. We all knew that so lets move on.

The error that we face tells us something like this:
~~~bash
// ... Some unneeded clutter above ...
System.InvalidOperationException: A second operation was started on this context instance before a previous operation completed. This is usually caused by different threads
concurrently using the same instance of DbContext. For more information on how to avoid threading issues with DbContext, see https://go.microsoft.com/fwlink/?
linkid=2097913.                 
    at Microsoft.EntityFrameworkCore.Infrastructure.Internal.ConcurrencyDetector.EnterCriticalSection()
    at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(IList`1 entriesToSave, CancellationToken cancellationToken)
    at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChangesAsync(StateManager stateManager, Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
    at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
    at Microsoft.EntityFrameworkCore.DbContext.SaveChangesAsync(Boolean acceptAllChangesOnSuccess, CancellationToken cancellationToken)
    at ParallelEfContext.Controllers.LetterController.<BadParallel>b__5_0(Char letter, CancellationToken cancellationToken) in C:\Users\nikol\source\repos\ParallelEfContext\Controllers\LetterController.cs:line 59
    at System.Threading.Tasks.Parallel.<>c__53`1.<<ForEachAsync>b__53_0>d.MoveNext()
    --- End of stack trace from previous location ---
    at ParallelEfContext.Controllers.LetterController.BadParallel(Char[] letters) in C:\Users\nikol\source\repos\ParallelEfContext\Controllers\LetterController.cs:line 50
~~~
When reading the error message the problem is obvious. The DbContext wasn't meant to be used like this. But we *reeeeeeeeeeeeeally* need this to work so how can we do it?

## Attempt 1: Mutex
One would think that having a `Mutex` would save you and while that is true to some extent, you probably lose a lot. I've been playing a lot with [Rust](https://www.rust-lang.org/) and I've been coding for about a year and a half and one thing that I got from it is that if I am using mutexes I am losing a lot and I don't know how to actually solve the problem at hand. 

Looking at the code we would have to write to **actually** get the similar output we did in the [sequential approach](#sequential-approach) we would need to write something like:
~~~c#
[HttpPost("/mutex")]
public async Task<IActionResult> Mutex(char[] letters)
{
    var mutex = new Mutex();
    await Parallel.ForEachAsync(letters, async (letter, cancellationToken) =>
    {
        var threadId = Guid.NewGuid();

        mutex.WaitOne();
        await _appDbContext.AddAsync(new Letter
        {
            Char = letter
        }, cancellationToken);

        await _appDbContext.SaveChangesAsync(cancellationToken);

        _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", threadId, string.Join(", ", _appDbContext.Letters.Select(x => x.Char))));
        mutex.ReleaseMutex();
    });
    return Ok("Done");
}
~~~
One could argue that we could've released the mutex after each `DbContext` action and have code that looks like:
~~~diff
[HttpPost("/mutex")]
public async Task<IActionResult> Mutex(char[] letters)
{
    var mutex = new Mutex();
    await Parallel.ForEachAsync(letters, async (letter, cancellationToken) =>
    {
        var threadId = Guid.NewGuid();

        mutex.WaitOne();
        await _appDbContext.AddAsync(new Letter
        {
            Char = letter
        }, cancellationToken);
+       mutex.ReleaseMutex();

+       mutex.WaitOne();
        await _appDbContext.SaveChangesAsync(cancellationToken);
+       mutex.ReleaseMutex();

+       mutex.WaitOne();
        _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", threadId, string.Join(", ", _appDbContext.Letters.Select(x => x.Char))));
        mutex.ReleaseMutex();
    });
    return Ok("Done");
}
~~~
If this is added then the code will execute more randomly and the result may differ from execution to execution but at the end of the day since IO is so slow it will result in three log messages that print all the same letters meaning that we have handled the writes and reads correctly but are limited with database itself. It didn't really surprise me since databases are always the bottleneck of the system. What I really don't like about this approach is that we are adding a lot of code to a simple feature. If the feature was a reallife requirement then maybe this is justifyable but at the end imagine refactoring code and adding/removing these waits and releases... A sure deadlock bug will hapen at one point.

## Attempt 2: Not using context from DI and creating new connection each time

We are entering the area of some real improvements first in the quality of code, and on the performance side of things. This is the first thing that pops to mind right after the `Mutex` zone (or maybe it was your number one go and it was just me who really thought about mutexes at one point 🤣). The code looks like this:
~~~c#
[HttpPost("/new-connection")]
public async Task<IActionResult> NewConnection(char[] letters)
{
    await Parallel.ForEachAsync(letters, async (letter, cancellationToken) =>
    {
        var threadId = Guid.NewGuid();
        var currentContext = new AppDbContextFactory().CreateDbContext(new string[] { });

        await currentContext.AddAsync(new Letter
        {
            Char = letter
        }, cancellationToken);

        await currentContext.SaveChangesAsync(cancellationToken);

        _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", threadId, string.Join(", ", currentContext.Letters.Select(x => x.Char))));
    });
    return Ok("Done");
}
~~~
What I love about this is that really we didn't add anything special. If we didn't totally rename the variable name to `currentContext` we could've just not resolve `_appDbContext` from previous examples and just create it in `Parallel.ForEach` arrow function. To be fair this solutions suffers from the same issue the previous attempt had and that is the speed of IO meaning that the resulting log messages are again all letters printed for all three threads.
~~~bash
// log messages
[Thread 3528f312-a840-4deb-b6fc-b0609afd5324]: Current elements -> a, b, c, d, e, f, z, y, x
[Thread f7f12ba2-3426-4b29-813a-4a316887aa6b]: Current elements -> a, b, c, d, e, f, z, y, x
[Thread f260c359-912a-4375-939c-d559cb845420]: Current elements -> a, b, c, d, e, f, z, y, x
~~~
There is one thing that I don't like about this solution and what actually got me interested in this topic. Earlier in this post I've shown code for registering the `DbContext` as pooled meaning that we have 1024 cached connections just sitting around waiting to be used. If we create new connections we still have a lot of idle connections that could be used for this same thing! So how can we do that?

## Attempt 3: Using IDbContextPool

This has to be the most amazing find (atleast for me in dotnet EF). Now the code that I am going to share with you isn't something Microsoft ties to maintaining. They themselves state that the code could be changed and implementations should tie to it. But we always liked living on the edge and therefore I present to you the optimal solution that uses cached connections from DI container and that properly restores them and waits for them if all are in use meaning that no new connections are spawned and database shouldn't experience any more heardle than before:
~~~c#
private readonly ILogger<LetterController> _logger;
private readonly IDbContextPool<AppDbContext> _dbContextPool;
/// constructor to resolve deps
public LetterController(ILogger<LetterController> logger, IDbContextPool<AppDbContext> dbContextPool)
{
    _logger = logger;
    _dbContextPool = dbContextPool;
}

[HttpPost("/pool")]
public async Task<IActionResult> Pool(char[] letters)
{
    await Parallel.ForEachAsync(letters, async (letter, cancellationToken) =>
    {
        var threadId = Guid.NewGuid();
        IDbContextPoolable poolable = null;
        while (poolable == null)
        {
            poolable = _dbContextPool.Rent();
        }

        if (poolable is AppDbContext currentContext)
        {
            await currentContext.AddAsync(new Letter
            {
                Char = letter
            }, cancellationToken);

            await currentContext.SaveChangesAsync(cancellationToken);

            _logger.LogInformation(string.Format("[Thread {0}]: Current elements -> {1}", threadId, string.Join(", ", currentContext.Letters.Select(x => x.Char))));
        }

        await _dbContextPool.ReturnAsync(poolable);
    });
    return Ok("Done");
}
~~~
The awesome thing about this is that the cached connections are enough to bring back the randomness in log messages to, giving us an output similar to the one we've seen while executin the code sequentionally. 
~~~bash
// log messages
[Thread e87cda14-f103-4984-98aa-7588aa7ade44]: Current elements -> a, b, c, d, e, f, y
[Thread a1525159-8bb0-49fc-bc82-2b1ba3ccad41]: Current elements -> a, b, c, d, e, f, y, x
[Thread 5008cbed-e1f0-4e47-aa4e-f055f212fbc2]: Current elements -> a, b, c, d, e, f, y, x, z
~~~

## Conclusion
This was a fun little exploration about how can we achieve a pretty edge case scenario and still get away with maintaining the code readable and efficient. In my opinion databases are the biggest and most cherished thing we own in a system and we should try and use it efficiently. I've seen some projects that couldn't achieve the desired performance with the highest paying subscription for databases on cloud and that wasn't because their usecases were hard or difficult but rather due to the lack of expertise and lack of this kind of exploration. 
> Should they go and optimize their code to use something like this in a lot of places and be done with it?   

Probably no. This isn't a silver bullet and things like this are a mere trick to squeeze the last 5-10% if possible. I would like to explore dealing with big data systems and optimizing database access on a lot of levels and I will for sure write a post about that if I do.

Until then, cheers!