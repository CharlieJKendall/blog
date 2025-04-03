---
layout: post
title: "Why you don't need to use cancellation tokens in .NET"
date: 2024-05-27
tags: [ 'dotnet', 'C#', 'CancellationToken', 'ASP.NET' ]
summary: 'There are a lot of good reasons why you might want to avoid handling request aborts in your application code. This post runs through a few of the key downsides to passing cancellation tokens around your ASP.NET application.'
---

I recently had a good ol' fashioned debate with some colleagues regarding usage of `CancellationToken` in ASP.NET Core web applications. They argued that it is a best practice to include a `CancellationToken` parameter on each of our endpoint method signatures, and propagate the value throughout our codebase to ensure it is eventually passed to any cancellable operations. Although it is an entirely valid position to hold, I think there are several aspects that may have been neglected when considering whether to head down that road.

Having now spent far more time thinking about cancellation tokens than anybody should reasonably ever have to, I've split my thoughts out into three separate posts:

1. Why you don't need to use cancellation tokens in .NET (this post)
2. Why cancellation tokens are great!
3. How to use cancellation tokens cleanly in .NET

## Setting the scene

If you're already familiar with how the `CancellationToken` struct works and the purpose it serves, then feel free to skip this section. [The Microsoft docs summarise](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken) the intended usage nice and succinctly: "Propagates notification that operations should be canceled". The object itself is essentially a glorified boolean value representing the intent to cancel operations within a particular scope, where the scope is usually defined by the code that creates the cancellation token. There are three main ways by which you can interact with cancellation tokens:

- Examining the `IsCancellationRequested` property. This returns a boolean value that represents the current intent for the token - `true` if cancellation has been requested, otherwise `false`.
- Invoking the `Register()` method. There are several overloads for this method, all of which take an `Action` parameter. If the token is cancelled, the `Action` you provide will be invoked.
- Invoking the `ThrowIfCancellationRequested()` method. This method simply throws an `OperationCanceledException` if the `IsCancellationRequested` returns `true` at the point that this method is invoked.

The scope in question throughout the discussion with my colleagues was an HTTP request, so this is what I'm largely going to focus on in this series. You can access the `CancellationToken` associated with an HTTP request directly via the `HttpContext.RequestAborted` property, or more commonly by adding a `CancellationToken` parameter to your endpoint method signatures or minimal API endpoint delegates like below:

### Controller endpoint

``` cs
[HttpGet]
public ActionResult<GetBlogPostsResponse> GetBlogPosts(CancellationToken cancellationToken)
{
    return _blogService.GetBlogPosts(cancellationToken); // Use the CancellationToken
}
```

### Minimal API endpoint

``` cs
app.MapGet(
    "/BlogPosts",
    (CancellationToken cancellationToken, [FromServices] IBlogService blogService) =>
    {
        return blogService.GetBlogPosts(cancellationToken); // Use the CancellationToken
    });
```

## That sounds useful, why wouldn't I want to do that?

Well... it is useful! But there's a second half to that equation we need to consider carefully here: why _would_ I want to do that? It is all too common for people to put unequal weighting on one side of the scales when evaluating whether to use a particular technology, library, or API. This is especially true when it has been branded as a 'best practice' by those whom they respect the opinion of. One of the skills that is all but essential to be a great engineer is the ability to approach both sides pragmatically and with minimal prejudice, making an effort to avoid being blinded by the shiny new framework or the latest trend. So, with that being said, here are my thoughts on why you _might_ want to think twice before jumping on the cancellation token bandwagon:

### 1. It's not 'request cancellation', it's 'request aborted' and is therefore unreliable

The `HttpContext.RequestAborted` cancellation token is cancelled when the underlying TCP connection between the client and your application is **aborted**. This can be due to the client explicitly disconnecting, but could just as easily be due to an interruption in the network somewhere between the client and server. Your application is unable to differentiate between the various reasons for the connection being aborted; all it knows is that the connection is no longer open. Networks are inherently unreliable, so you can end up in a multitude of situations whereby the client and server have misaligned understandings of the current state of affairs - perhaps your client disconnected, however the `FIN` packet never made it to your server so it continues to execute the request to completion only to get an error when it attempts to send a response back to the client.

For me, this is one of the most important points to be clear on in order to frame a discussion properly. Your application is **not** able to make decisions based on whether a client has cancelled a request, it is only able to make decisions based on whether it thinks the underlying TCP connection has been aborted or not.

### 2. Do you _actually_ need to handle aborted requests?

It may sound obvious, but more often than not the answer to this question is "no". Clearly _your_ answer is going to vary depending on your particular circumstances, but I'd hazard a guess that many backend applications aren't _too_ bothered about explicitly cancelling their outbound HTTP requests. So if you have an internal API that nobody is interested in cancelling their requests to and you have no need to worry about other types of abort, then why should you go to the effort of implementing and maintaining that functionality? It may sound lazy, but you can always reconsider your decision if there comes a time in the future when handling cancellation is justifiable, so for now it may be worth avoiding a bit of complexity and moving faster.

When it comes to HTTP requests made to your application from **web browsers**, the story goes a little differently. Navigating around your browser causes certain requests to the server to be cancelled by the client, for example the default behaviour of many browsers is to cancel incomplete `XMLHttpRequests` when you navigate away from the page that initiated them. So if you have a browser facing application, you're going to be seeing request aborts whether you like it or not! However, the question still stands: do you actually need to handle them? What would happen if you didn't? As with all code, there is an associated maintenance overhead and time cost - so if you have no hard data to justify doing so then it might be worth leaving it for the future!

### 3. What do you do when a request is aborted _after_ it has modified some persisted state?

An example will go a long way here, so let's say you have an endpoint that adds a new credit card to a user's account. The order of operations may look something like this:

1. Insert the credit card details into the 'CreditCards' table in a relational database
2. Update the relevant 'UserDetails' document in your document database to contain the inserted credit card's primary key value

Both of these steps persist data over a network connection and can therefore fail for a variety reasons. Network failures are an inescapable reality of software development, and should always be considered when designing your own systems. In the case of our endpoint outlined above, an expected failure mode is the application being unable to connect to the document database _after_ having successfully inserted a row into the relational database. This situation would typically lead to an exception being thrown by the library code that we use to interact with our database. When considering how to handle this, the following questions might come to mind:

- Should we return a 500 response to the user?
- Should there be a compensating action that removes the newly added row to the 'CreditCards' table?
- Does the compensating action happen within the scope of the failed HTTP request?

As you can see, there is a fair bit to consider if we want to implement this in a way that avoids data corruption. However, before jumping straight in to a solution, we should pause and ask ourselves two questions: **"how often will this type of failure happen?"** and then **"if it does happen, what is the impact?"**. Most of the time, the answers will help guide you towards an informed decision as to whether it is worth trading off the time cost and additional code complexity in order to have a more resilient system. In practice, utilizing high availability databases/services along with basic resiliency policies mitigates most of the risk associated with the 'how often?' question in this scenario. This largely removes it from the equation, putting a **much larger** emphasis on the 'what is the impact?' side of things. In a perfect world, we'd be implementing the perfect solution - but we don't live in a perfect world, so trade-offs must be made. It's unlikely that you're writing code that accounts for cosmic ray bit flipping, or even your application process being terminated unexpectedly mid-execution; and that's fine, because they're sufficiently rare to not be worth investing your time into accounting for.

Sure that's interesting and all, but how is this relevant to handling aborted requests?!

In brief, handling aborted requests **drastically** changes the answer to 'how often will this happen?', meaning many of those error modes you'd previously written off as 'it almost never happens' will suddenly be happening a whole lot more frequently! To understand why this is the case, we'll need to look at the types of operation you might want to cancel as well as understanding how libraries tend to implement cancellation.

#### What sort of operations should be cancellable?

There are many reasons for wanting to support cancellation of an operation, but the most common by far is because the operation itself is expensive. This expense is _usually_ related to time cost, but can also relate directly to raw monetary cost. Your code could be making a request to a third party API that charges per request, so cancelling execution before it hits this code path will directly save you money. Going back to the time cost aspect, you've got two main ways that can result in you racking it up:

- **Compute** -> This type of work presents in many ways, but almost always involves loops with large numbers of iterations _or_ very complex calculations. For example determining the nth digit of pi, or applying a filter to a large image
- **I/O** -> Serializing/deserializing data to and from bytes takes time, as does sending data over networks and reading from disks. For example sending a query to a relational database and receiving the results, or writing a large document to a filesystem

In the world of C#, you'll notice that the **vast majority** of well-maintained APIs and libraries have overloads for their I/O and compute bound methods that support cancellation via a `CancellationToken` parameter. This includes all the methods that you're using to send queries to your document database in the scenario above!

#### How do libraries implement cancellation?

Most libraries and APIs take the sensible opinion that they **must** communicate when their code has attempted cancellation back to the caller. A very small minority of these return this information as part of a return value or even an `out` parameter, with most libraries opting to throw an `OperationCanceledException` instead. Putting this into the context of our scenario with the document database, the main difference between you **cancelling a query** and it **failing due to a network issue** is the type of exception that gets thrown from within the library you're using to interact with the database! For all intents and purposes, these situations should be handled in the same way by your application code.

So to summarise: if you go down the path of handling aborted requests then you may well need to reconsider many of the pre-existing solutions you've implemented based on the assumption that certain network related failure modes are rare enough to not be a problem.

### 4. It can add complexity and maintenance overhead to your codebase

Armed with the knowledge we now have, it's starting to sound a lot like every single `async` method in our codebase is going to need a `CancellationToken` parameter because it eventually calls into an I/O or compute bound API... right? Well not necessarily, but that is unfortunately how it is so often implemented! What's worse is that these parameters are sometimes made optional in codebases, making it all too easy for callers to accidentally forget to pass through a token thus going directly against the cooperative mechanism by which the tokens operate. Alas, all is not lost as I dive into a cleaner and less invasive opt-in pattern in the third post in the series! Yet even with a cleaner implementation, it's still additional code that needs to be written, tested, and maintained.

## Summary

I've discussed a number of points that make a case for handling request aborts only when it becomes necessary and justifiable, challenging the view that it is a best practice to handle them by default. Implementation does not come for free, so it's important to weigh up the pros and cons before committing to it. In my next post, I take a look at the other side of the coin and run through a number of wins that handling request aborts can give us.
