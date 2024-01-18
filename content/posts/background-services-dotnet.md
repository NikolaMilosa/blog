---
title: "Long running jobs in ASP.NET"
date: 2024-01-18T12:58:31+01:00
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

## Defining the problem

So lets say they wanted to migrate their code into the app? How would they do that? What they could use (at least in asp.net core) is [background workers](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-7.0&tabs=visual-studio). There is a number of things these are useful for. What we want to do is be able to trigger a with an API request which would start the process and return *Ok* possibly with some more data. It could return the identifier of a job so the client can poll the API to see if the job is finished.

## How I did it

### Background Worker

First of all here is the code:
```c#
using System.Threading.Channels;

class BackgroundGenerator<T> : IHostedService
{
    private readonly ILogger<BackgroundGenerator<T>> _logger;
    private readonly ChannelReader<T> _requests;
    private readonly int _maxBatchSize;

    public BackgroundGenerator(ILogger<BackgroundGenerator<T>> logger, ChannelReader<T> requests, int maxBatchSize)
    {
        _logger = logger;
        _requests = requests;
        _maxBatchSize = maxBatchSize;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        Task.Run(async () => await RunThread(cancellationToken));
    }

    private async Task RunThread(CancellationToken cancellationToken) {
        // Normal work, until the API is running
        _logger.LogInformation("Starting execution of background service...");

        while(await _requests.WaitToReadAsync(cancellationToken)) {
            _logger.LogInformation("Received data, creating a batch...");
            var batch = new List<T>();
            while(_requests.TryRead(out T item)){
                batch.Add(item);
                if (batch.Count == _maxBatchSize) break;
            }
            await ExecuteLogic(batch);
        }
    } 

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Received stop signal, waiting for all batches to process...");
        // Either save the status of Queue in some file so when the service restarts if the work is important
        // Or just exit
        _logger.LogInformation("Stopping generator...");
        return Task.CompletedTask;
    }

    private async Task ExecuteLogic(List<T> batch) {
        foreach(var item in batch) {
            _logger.LogInformation($"Processing item [{item}]");
            await Task.Delay(1000);
        }
    }
}
```

I wanted to achieve a couple of things:
1. I wanted to use [channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels). I needed the architecture of single consumer and many producers where potentially each request will be a producer and only one background service will be a consumer.
2. I wanted to make it generic. Imagine that each `T` is a request that was created and a background service of type `BackgroundGenerator<T>` can execute it. With a bit more work this could've been abstracted a bit more and we would have each service do one thing. For the demo purpose it was not done since I just wanted to print it out.
3. I wanted to have batch processing. This is a way to optimize the workload. Each request could've been handled separately but with this approach we could've bundle some requests together to squeeze out more performance from each iteration.

The whole class could've been made abstract with a requirement to implement only a `ExecuteLogic(List<T> batch)` and we would've have a nice template design.

### Registering everything up

To wire everything up correctly in the `Program.cs` I needed to do a couple of things.

First thing was to create a channel.
```c#
var channel = Channel.CreateUnbounded<Guid>(new UnboundedChannelOptions {
    SingleReader = true,
    SingleWriter = false,
});
builder.Services.AddSingleton(channel);
```
I chose to go with an `Unbounded` channel. The difference is if the channel has size or is it dynamically changed. If I went with the approach of using a `Bounded` channel I could've implemented batching of jobs with that. Where I would set the batch size as a bound and then execute the whole channel at once. However for our logic it would break since its possible that our writer can't write to the channel making it block and not returning any answer to client which is not ideal.

### Testing the use-case

The code for the incomming request is simple
```c#
app.MapGet("/process", async ([FromServices] Channel<Guid> channel, uint num) => {
    for (int i = 0; i < num; i++) {
        await channel.Writer.WriteAsync(Guid.NewGuid());
    }

    return "Ok";
})
.WithName("Process")
.WithOpenApi();
```

It accepts amount of writes to add to the channel. When curled with a get request
```bash
curl http://localhost:5122/process?num=<num>
```
It should start the working and the output will look somewhat like this:
```log
info: BackgroundGenerator[0]
      Received data, creating a batch...
info: BackgroundGenerator[0]
      Processing item [55a45875-bebd-4d9c-9c4e-4001fce8e0ae]
info: BackgroundGenerator[0]
      Processing item [b6d6e419-3cc5-4059-95bb-710a7d6b9801]
info: BackgroundGenerator[0]
      Processing item [e818767a-23b3-4497-a3e6-aa4cd74a0af7]
info: BackgroundGenerator[0]
      Processing item [d14239d6-d770-49dd-bba7-3de65cd6099e]
info: BackgroundGenerator[0]
      Processing item [cff3ad50-5699-4fd1-8c37-0065b179473b]
info: BackgroundGenerator[0]
      Received data, creating a batch...
info: BackgroundGenerator[0]
      Processing item [17454bf7-18fe-4461-a2e2-2a8899fc2c04]
info: BackgroundGenerator[0]
      Processing item [9420f5a2-f5fd-4efd-94b9-6eb7c271857a]
info: BackgroundGenerator[0]
      Processing item [71160d5e-cfa6-4e7c-a2c4-670f43dd5c1a]
```

## Conclusion

I quite like what this can do and how extensible it is. One has to remember that if we need to resolve scoped dependancies from DI container we need to manually create a scope for our background service.
```diff
...
    private readonly ILogger<BackgroundGenerator<T>> _logger;
    private readonly ChannelReader<T> _requests;
    private readonly int _maxBatchSize;
+   private readonly IServiceScope _scope;
+   private readonly DbContext _db;

-    public BackgroundGenerator(ILogger<BackgroundGenerator<T>> logger, ChannelReader<T> requests, int maxBatchSize) {
+    public BackgroundGenerator(ILogger<BackgroundGenerator<T>> logger, ChannelReader<T> requests, int maxBatchSize, IServiceScopeFactory scopeFactory) {
        _logger = logger;
        _requests = requests;
        _maxBatchSize = maxBatchSize;
+       _scope = scopeFactory.CreateScope();
+       _db = _scope.ServiceProvider.GetRequiredService<DbContext>();    
    }

...
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Received stop signal, waiting for all batches to process...");
        // Either save the status of Queue in some file so when the service restarts if the work is important
        // Or just exit
+       _scope.Dispose();
        _logger.LogInformation("Stopping generator...");
        return Task.CompletedTask;
    }
...
```
Now as always the question is when to use these? I say that people should be very careful since usually extracting into lambda/functions is done to tackle the wast amount of resources that these jobs can take. If servers are strong and sit around 30-40% of workload they can easily host some logic within them. All this falls into water if the logic cannot be or can hardly be coded in .net. As I said in the beginning of this post I am not sure how I feel about a language trying to tackle all the problems at once and sometimes its just better to use a different technology. Although this can be coupled with some compiled language (e.g. rust) and having background jobs that run the rust binary which are known to squeeze the last bits of performance when needed.

I've written enough about .net for now. Onto new advantures 🚀. 

Until then, cheers!