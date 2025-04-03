---
layout: post
title:  "Why cancellation tokens are great!"
date:   2024-06-05
tags: [ 'dotnet', 'C#', 'CancellationToken', 'ASP.NET' ]
summary: 'There are a tonne of great reasons to take respect operation cancellation using the CancellationToken struct in dotnet. Here is my take on why and when you could consider using them!'
---

My previous post in my mini series about cancellation tokens was about how I often see developers using `CancellationTokens` in their ASP.NET web applications without considering the trade-offs of doing so. They have often jumped on the bandwagon and assumed it to be a 'best practice' without necessarily asking themselves _why_ they are using them and considering the additional overhead that they add 🤠

Yet all is not bad in the world of `CancellationToken`, and there are many valid reasons to take advantage of them! This second post focuses on some of the key arguments **for** handling request aborts and cancellations within your applications.

1. Why you don't need to use cancellation tokens in .NET
2. Why cancellation tokens are great! (this post)
3. How to use cancellation tokens cleanly in .NET

### 1. It can save you precious money 💰

As I mentioned in my previous post, there are two main costs that can be saved through cancellation of unnecessary operations: **time** and **raw monetary value**. In fact, time cost _often_ ends up translating to monetary value in some form or other; executing code requires hardware and power, which rarely come for free! On a lesser scale these costs may be insignificant, but when you're serving millions of requests per day these small numbers can start to add up rather fast. This is more true than ever if clients to your application are using [more aggressive resiliency strategies such as hedging](https://www.pollydocs.org/strategies/hedging.html), resulting in a greater number of requests being explicitly cancelled by the client.

If you aren't already handling request aborts and you're looking to do so in the interest of **cost savings**, then my advice would be first to measure before committing to implementation. This could be as simple as a piece of middleware that registers an `Action` with the `HttpContext.RequestAborted` token to increment a `Counter` metric whenever a request is cancelled:

``` cs
public class RequestAbortedLoggingMiddleware
{
    private static readonly Meter _meter = new Meter("Application.Metrics", "1.0.0");
    private static readonly Counter<long> _requestsAbortedCounter = _meter.CreateCounter<long>(
        "requests.aborted",
        "requests",
        "Total number of requests aborted.");

    private readonly RequestDelegate _next;
    private readonly ILogger<RequestAbortedLoggingMiddleware> _logger;

    public RequestAbortedLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        context.RequestAborted.Register(() => _requestsAbortedCounter.Add(1));

        await _next(context);
    }
}
```

### 2. If it hurts, do it more often

I am a big advocate for DevOps practices, which this point is a (not so) subtle nod to. In the world of DevOps, just because something is difficult or painful does not mean you should avoid it; counterintuitively, you should do it more frequently. Oftentimes these scenarios are not only inevitable, but critical to get right: application deployments, database migrations, and infrastructure provisioning just to name a few. It's not too far of a stretch to extend that same logic to handling request aborts - handling these aborts in your application code won't usually add _new_ failure modes, rather it will make certain pre-existing ones more prevalent.

It's fairly typical for code that interfaces with other systems over a network to throw an exception when something goes wrong at the network level. Much of the time there is no easy way to recover from the failure, meaning that the result of the network request will not be available to your application. These exceptions can be caused by things like timeouts and network blips... or maybe, your application intentionally terminating the connection because it no longer needs the query result?

The more time you spend thinking about and handling these types of failure, the easier it will become as time goes on. You'll get better at identifying different types of problem and develop improved ways of dealing with them, all while building an application that is **much** better versed at keeping out the chaos that the real world throws at us.

### 3. There _are_ some quick wins

Having said all that I did about the difficulties surrounding maintaining data integrity in the previous article, there is a large cohort of requests that this logic does **not** apply to: pure queries. For this type of request, there are **no side effects** and is therefore no need to worry about compensating actions or corrupted state; you can freely cancel any in-progress work without so much as a second thought. Well, almost... these cancellations _will_ result in `OperationCanceledException` being thrown, so you may need to tweak your monitoring and error tracking to prevent these expected cancellations from wreaking havoc with your dashboards and alerting! You may have other minor considerations to work out, but pure queries are certainly a solid place to start playing around if you're going to go down the request abort handling route.

### 4. It can make your application more resilient

Let's say you have an endpoint that is _particularly heavy_ on compute, for which the algorithmic time complexity is a linear function of the amount of data the client requests from it - in terms of Big O, this is O(n). In other words, requesting 100 pieces of data takes 100 times longer to respond than it does for 1 piece of data. Let's _also_ say you care about the user experience of you frontend client, and implement logic to automatically retry any `XMLHttpRequest` `GET` that does not complete within 5 seconds. This is not an uncommon strategy in some form or another, and one that can do tragic things when you have longer-running requests and your server doesn't respect request cancellation.

With this stage set, consider what happens when a client hits your endpoint and requests enough data for it to take 60 seconds to complete (and prepare yourself for some pain). The client will cancel the initial request after 5 seconds, and fire another one off. Meanwhile your server _continues_ to process the first request whilst graciously accepting the second... and the third... and the fourth... until it's processing 12 computationally heavy requests from a single client simultaneously and **indefinitely until the user navigates away from the page** 😬

Believe it or not, I have witnessed an incident caused by a situation very similar to the one described above! Despite application autoscaling policies kicking in, this still red barred the CPU of every instance of the backend service that was serving the requests. The learning we took: handle the cancellation of longer running requests on the server, provide a way to opt out of client side retry policies, and continue to ensure excellent observability of all applications.

## Summary

There are many great reasons to make the effort to handle request aborts in your application code. If you're not currently handling them but are considering doing so, it's a good idea to try and measure the impact before jumping into implementation. In the final post in the series, I'll run through a clean opt-in implementation that is easy to introduce to an existing codebase.
