---
layout: post
title:  "How to use cancellation tokens cleanly in .NET"
date:   2024-06-18
tags: [ 'dotnet', 'C#', 'CancellationToken', 'ASP.NET' ]
summary: 'Rather than passing CancellationToken parameters around your codebase, consider using the ambient context pattern to keep things clean. In this post, I run through a simple implementation to get the ball rolling.'
---

This is the third and final part of my ramblings relating to usage of the `HttpContext.RequestAborted` cancellation token in ASP.NET web applications. In this post I run through a simple implementation that allows us to avoid much of the mess caused by passing `CancellationToken` parameters everywhere whilst providing an opt-in/out mechanism for handling cancellation.

1. Why you don't need to use cancellation tokens in .NET
2. Why cancellation tokens are great!
3. How to use cancellation tokens cleanly in .NET (this post)

### Introducing the ambient context pattern

One way that information gets passed around your codebase is by explicitly doing so using method parameters. This makes total sense when viewing any given method within the unit of execution in which it exists - for example, a `GetBlog()` method needs to explicitly take in a blog identifier parameter in order to be able to get the appropriate blog. Let's say the `GetBlog()` method _also_ needs to carry out some level of authorization to prevent a blog being returned if the current user does not have permission to view it, for which it therefore needs an understanding of the 'current user'. Does the data needed to determine this also get passed in as a parameter? Maybe, or maybe not... this could be a great opportunity to use ambient data to keep things clean!

Instead of explicitly passing information around via parameters, we could choose to store this data in a shared location that the code can access implicitly whenever it needs to do so. This could be as simple as a static property, or more commonly an injected dependency scoped to a particular context of execution. Going back to our `GetBlog()` method example, we may create some sort of 'authorization context' that is injected into our `BlogService` which can be accessed from wherever authorization is necessary within the service. We're essentially treating the resource authorization as an implementation detail that the direct caller doesn't need to worry itself with.

### Now, back to cancellation tokens

Depending on where our `GetBlog()` data access method is being called from, there may be several different ways that cancellation can be triggered. It's likely that most, if not all, of the callers don't concern themselves with creating or working out where to find the relevant cancellation token; they get passed a token from _their_ callers that they simply pass through to the layer below:

``` cs
// BlogController.cs
public async Task<BlogDto> GetBlog(string blogId, CancellationToken token)
{
    return await _service.GetBlog(blogId, token);
}

// BlogService.cs
public async Task<BlogDto> GetBlog(string blogId, CancellationToken token)
{
    var blog = await _repository.GetBlog(blogId, token);
    return Map.ToDto(blog);
}

// BlogRepository.cs
public async Task<Blog> GetBlog(string blogId, CancellationToken token)
{
    return await _dbContext.Blogs.FindAsync(blogId, token); // Third party library call
}
```

It's a simple example and doesn't look too awful, but in the real world you'll find that nearly every `async` method in your codebase ends up taking a `CancellationToken` parameter if you implement it like this. Let's take a step back and make some high level observations, then we'll hopefully start to see how using an ambient context could help us keep things much cleaner:

- The `CancellationToken` associated with the current request (`HttpContext.RequestAborted`) is set very early on in the request pipeline, before your controller method is invoked
- Our application code does nothing with the token other than pass it through to `async` methods it calls

With this in mind, could we cut out the middlemen and implement a way for the code that actually _uses_ the token to grab it at the point that it needs to do so? Yes, we absolutely can - by taking advantage of the ambient context pattern 🙌 Let's update the previous code snippet to follow this pattern. Note how there are no longer any cancellation tokens being passed through layers; it is retrieved from the context _only_ at the point that it is required. Looks cleaner already!

``` cs
// BlogController.cs
public async Task<BlogDto> GetBlog(string blogId)
{
    return await _service.GetBlog(blogId);
}

// BlogService.cs
public async Task<BlogDto> GetBlog(string blogId)
{
    var blog = await _repository.GetBlog(blogId);
    return Map.ToDto(blog);
}

// BlogRepository.cs
public async Task<Blog> GetBlog(string blogId)
{
    return await _dbContext.Blogs.FindAsync(blogId, _cancellationContext.Token); // Third party library call
}
```

### Implementing our cancellation context

We need a simple abstraction that provides a way for our code to retrieve a `CancellationToken` for the current scope **and** a way for our application to set this token to the correct value as early as possible in the lifetime of that scope. Setting this value is going to be a **lot** less common than getting it, so let's draw a clear distinction between these two functionalities with our abstractions to help avoid accidental misuse in the future.

``` cs
// Used by most of the code to retrieve the token
interface ICancellationContext
{
    CancellationToken Token { get; }
}

// ONLY used by the code that needs to set the value
interface ISettableCancellationContext : ICancellationContext
{
    new CancellationToken Token { get; set; }
}

// The concrete implementation
class CancellationContext : ISettableCancellationContext
{
    public CancellationToken Token { get; set; }
}
```

Now we've got a class and some interfaces that represent the abstraction we'd like to manage cancellation with, we need to register them with our dependency injection container. As we're concerned with the cancellation of a particular scope, these will need to be registered with a `Scoped` lifetime. We also need to make sure that the same scoped `CancellationContext` instance is resolved regardless of whether we resolve `ICancellationContext` or `ISettableCancellationContext` from the dependency container:

``` cs
services.AddScoped<ISettableCancellationContext, CancellationContext>();
services.AddScoped<ICancellationContext>(x => x.GetRequiredService<ISettableCancellationContext>());
```

On to the final piece of the puzzle: how to set the context's token in the first place. This is where **you** need to make the decision about what the default behaviour of your application is going to be, and whether to provide a way to opt in or out to request abort cancellation. You have three options, with the first one being _my_ preferred route:

- **Never trigger cancellation when a request is aborted by default, with a way to opt in**
- Always trigger cancellation when a request is aborted
- Always trigger cancellation when a request is aborted by default, with a way to opt out

#### Never trigger cancellation when a request is aborted by default, with a way to opt-in

For the default case where cancellation shouldn't be triggered, there's nothing that needs doing; the default value of the `CancellationContext.Token` property is equivalent to `CancellationToken.None` so there is no need to worry about setting it. But how do we want to give ourselves the ability to opt in? There are a load of ways to go about this, but one of the cleaner ones would be to use action filter attributes. This way, all you need to do to opt in is decorate controller classes or endpoint methods with your attribute.

``` cs
// Action filter attribute definition
public class SupportsRequestCancellationAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        var cancellationContext = context.HttpContext.RequestServices.GetRequiredService<ISettableCancellationContext>();
        cancellationContext.Token = context.HttpContext.RequestAborted;
    }
}

// Usage on a controller endpoint
[ApiController]
[Route("[controller]")]
public class CancellationTokenController(IBlogRepository repository) : ControllerBase
{
    private readonly IBlogRepository _repository = repository;

    [SupportsRequestCancellation] // <------- Here's the magic
    [HttpGet("supports-cancellation")]
    public async Task<BlogDto> GetSupportsCancellation(string blogId) =>
        await _repository.GetBlog(blogId);

    // No attribute = cancellation is not supported
    [HttpGet("does-not-support-cancellation")]
    public async Task<BlogDto> GetDoesNotSupportCancellation(string blogId) =>
        await _repository.GetBlog(blogId);
}
```

#### Always trigger cancellation when a request is aborted

As before, there are many ways to achieve this. A simple one would be to add some middleware that pulls out the `RequestAborted` token from the `HttpContext` and sets the cancellation context:

``` cs
app.Use(
    async (context, next) =>
    {
        var cancellationContext = context.RequestServices.GetRequiredService<ISettableCancellationContext>();
        cancellationContext.Token = context.RequestAborted;

        await next(context);
    });
```

#### Always trigger cancellation when a request is aborted by default, with a way to opt out

I won't give an example of this one, as it's just going to be a combination of the previous couple of snippets using middleware to set the token and an action filter attribute to opt out.

### What about background workers and other execution scopes?

Just like with HTTP endpoints, the same pattern can be applied to any other execution scope that may want to handle cancellation. All you need to do is find a way to set the token on the cancellation context as early as possible in the lifetime of the scope, which the vast majority of frameworks will provide an easy way for you to do: Azure Functions support middleware pipelines, Mass Transit has consumer filters, background workers are passed a `CancellationToken` parameter.

### Summary

In this series of posts, I ran through the trade-offs that should be considered when deciding whether to handle the `HttpContext.RequestAborted` cancellation token within your applications. I run through the implementation of a simple ambient context-based abstraction that can be used to opt in to handling cancellation whilst simultaneously avoiding the cruft that cancellation token propagation typically adds to a codebase.

I've uploaded a fully working set of [examples to my GitHub here](https://github.com/CharlieJKendall/examples), just pop open the CancellationTokens.sln file in the repository root.
