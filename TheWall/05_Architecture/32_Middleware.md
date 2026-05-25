# Chapter 32 — Middleware: What It Actually Is

I have, over the years, asked a fair number of working ASP.NET developers to define *middleware*. The answers cluster into three groups. A small number give a precise, correct answer. A larger number give an answer that is roughly right but gestures vaguely at *"the pipeline"* and *"things that run before the controller."* And a surprising number, when pressed, reveal that they have written middleware, configured middleware, and debugged middleware without ever quite forming a mental model of what it *is*.

This chapter is for the second and third groups. It is short, because the concept is genuinely simple once stated plainly. The complexity people associate with middleware is almost entirely a fog that clears the moment the underlying shape is visible.

## What it actually is

Middleware is *a function that takes a request, optionally does something, and optionally calls the next function in a chain.*

That is the entire concept. Every piece of middleware in an ASP.NET application is, at heart, this shape:

```csharp
public async Task Invoke(HttpContext context, RequestDelegate next)
{
    // do something before the rest of the pipeline
    await next(context);            // call the next middleware (or the endpoint)
    // do something after the rest of the pipeline
}
```

`next` is the rest of the pipeline, packaged as a single callable thing. When you call `next(context)`, you are handing control to the next middleware in line; when that returns, control comes back to you for the *after* portion. The very last thing in the chain is the endpoint itself — your controller action or minimal-API handler.

## The Russian doll

The mental model that makes everything click is the *Russian doll* (the nesting matryoshka).

Each piece of middleware wraps the next one. The outermost middleware runs first and finishes last; the innermost (the endpoint) runs in the middle. A request entering the application travels *inward* through each layer, reaches the endpoint at the centre, and then the response travels *outward* through the same layers in reverse.

```
Request  →  [ Logging  →  [ Auth  →  [ Exception  →  [ ENDPOINT ] ]  ]  ]  →  Response
              run before    run before  run before      the centre
              ...           ...         ...
              run after  ←  run after ← run after  ←
```

This is why the *order* of middleware registration matters so much, and why it is the single most common source of middleware bugs. If you register authentication *after* the endpoint routing, authentication runs too late. If you register exception handling *inside* (after) the thing that throws, it cannot catch the exception. The order is the architecture.

```csharp
app.UseExceptionHandler();   // outermost — must wrap everything to catch everything
app.UseHttpsRedirection();
app.UseAuthentication();     // must run before authorisation
app.UseAuthorization();
app.MapControllers();        // innermost — the endpoints
```

Read that block top to bottom and you are reading the layers of the doll from outside to inside. The exception handler is registered first because it must be the outermost layer, wrapping everything else, so that an exception thrown anywhere inside it — including in the endpoint — travels back out to it.

## What middleware is for

Middleware is the right tool for *cross-cutting concerns that apply to many or all requests, at the HTTP level*. The canonical, legitimate uses:

- **Exception handling.** Catch unhandled exceptions anywhere in the pipeline, turn them into a sensible HTTP response.
- **Authentication.** Establish *who* the caller is, before anything that needs to know.
- **Authorisation.** Enforce *what* the caller may do.
- **Logging and request tracing.** Record the request, its timing, its correlation ID.
- **Response compression, CORS headers, HTTPS redirection.** Mechanical transformations of the HTTP envelope.
- **Rate limiting.** Reject too-frequent callers before they reach expensive work.

Notice the common thread: each of these operates on *the HTTP request as an HTTP request*, applies broadly across the application, and is genuinely orthogonal to any particular endpoint's business logic. That is the sweet spot.

## When the answer is "anything but middleware"

Middleware's breadth is also its trap. Because it sits across *every* request, it is tempting to reach for it for things that should be narrower, and the results are usually regrettable.

**Per-endpoint logic does not belong in middleware.** If a concern applies to one controller, or three, but not the rest, middleware is the wrong tool — it runs for everything, so you end up with `if (context.Request.Path.StartsWith("/api/orders"))` inside the middleware, which is middleware reimplementing routing, badly. The right tool is an *action filter*, an *endpoint filter*, or simply code in the endpoint.

**Business logic does not belong in middleware.** Middleware that reaches into the database, applies domain rules, and mutates business state has confused the HTTP layer with the application layer. The HTTP envelope is middleware's concern; the business meaning of the request is the endpoint's and the domain's. Middleware that knows what an `Order` is has reached too far inward.

**Anything that needs the routing result, the model-bound parameters, or the action context** is running too early for middleware in its classic position. By the time those exist, you are past the general pipeline and into filter territory. The filter pipeline (authorisation filters, action filters, result filters) exists precisely for concerns that need to know *which action* is about to run and *what its bound arguments are*.

The hierarchy, roughly: *middleware for the HTTP envelope and truly global concerns; filters for concerns that need the action context; endpoint code for the business logic itself.* Reaching for middleware when a filter or endpoint code would do is the most common middleware mistake, and it produces middleware that grows path-checking conditionals like barnacles.

## Writing one, properly

When middleware genuinely is the right tool, the modern convention is the factory-based form, which integrates cleanly with DI:

```csharp
public class RequestTimingMiddleware : IMiddleware
{
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(ILogger<RequestTimingMiddleware> logger)
        => _logger = logger;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var start = Stopwatch.GetTimestamp();
        try
        {
            await next(context);
        }
        finally
        {
            var elapsed = Stopwatch.GetElapsedTime(start);
            _logger.LogInformation("{Method} {Path} took {Elapsed}ms",
                context.Request.Method, context.Request.Path, elapsed.TotalMilliseconds);
        }
    }
}
```

Three things worth noting. The `try/finally` ensures the *after* portion runs even when something downstream throws — timing should be recorded whether or not the request succeeded. The dependencies arrive by constructor injection, the same as any other service. And the middleware does exactly one thing, at the HTTP level, for every request — which is the whole job description.

## Performance, in passing

Middleware runs on every request, which means a slow piece of middleware is a tax on the entire application, not just one endpoint. This cuts both ways.

A middleware that does cheap work — reading a header, stamping a correlation ID — costs essentially nothing and the breadth is a feature. A middleware that does *expensive* work on every request — a database lookup, a remote call, a heavy deserialisation — has quietly added that cost to every single endpoint, including the ones that did not need it. The health-check endpoint now pays for the per-request feature-flag lookup that only the order endpoints actually use.

The discipline: keep middleware cheap, because it runs for everything. Expensive work that only some requests need belongs further inward, where only those requests pay for it.

> *Putting an expensive database call in middleware is rather like making every visitor to the building remove their shoes at the front door on the off-chance one of them is heading to the room with the white carpet — thorough, undeniably fair, and the reason there is now a queue out into the car park.*

## The smells

- Middleware containing `if (context.Request.Path.StartsWith(...))` — it is reimplementing routing; you wanted a filter or endpoint code.
- Middleware that loads domain entities and applies business rules — the HTTP layer has reached into the application layer.
- Middleware registered in the wrong order — authentication after authorisation, exception handling not outermost.
- An expensive operation (DB call, remote call) in middleware that runs for every request, including ones that do not need it.
- Middleware that swallows exceptions silently instead of handling them visibly — the pipeline now fails in ways nothing records.
- A growing pile of bespoke middleware where a few well-placed filters would have been narrower and clearer.

## Recap

- Middleware is a function that takes a request, optionally acts, and optionally calls the next function in the chain.
- The pipeline is a Russian doll; registration order is the architecture.
- Middleware is for the HTTP envelope and truly global concerns — exceptions, auth, logging, compression, rate limiting.
- Per-endpoint logic wants a filter; business logic wants the endpoint and the domain. Reaching for middleware for these is the common mistake.
- Keep middleware cheap, because it runs for every request.

## Onwards

The next chapter takes a distinction that, once internalised, eliminates a remarkable amount of architectural dithering — the difference between a *service* and a *utility*, between the things that belong in the container and the things that are simply functions. It is a short chapter about a small idea with a large effect on the daily shape of a codebase.
