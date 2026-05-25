# Chapter 36 — Try/Catch: When To, When Absolutely Not To, and When You Should Be Ashamed

I warned you, at the end of the last chapter, that this one would raise its voice. It will. The exception is one of the most genuinely useful mechanisms in the language and one of the most consistently abused, and the gap between the two is wide enough that I am going to be, in places, slightly less even-tempered than the rest of the series has managed.

Let me state the central thesis at the top, in bold, because everything else follows from it. **Exceptions are for the exceptional. They are not flow control.** A great deal of bad exception handling traces directly to a failure to internalise that one sentence.

## What an exception is for

An exception signals that *something has happened which prevents the current operation from continuing, and which the current code does not know how to handle*. The database connection dropped. The disk is full. A bug produced a state the code's invariants said was impossible. These are *exceptional* — they are not the normal, expected outcomes of the operation; they are the breakdown of the conditions the operation assumed.

The mechanism is built for exactly this. An exception unwinds the stack, abandoning the failed operation, until it reaches a level that *does* know how to handle it — or, if no such level exists, the top of the application, where it is logged and the request fails cleanly. The unwinding is the point: it lets the code in between *not have to think about the failure at all*, because the exception flies past it to a handler that does.

## What an exception is not for

An exception is not for *expected* outcomes. *"The user typed an invalid email address"* is not exceptional — it is one of the most ordinary things a user can do, and a web form should expect it roughly half the time. *"The requested customer does not exist"* is not exceptional — it is a perfectly normal result of looking up an ID that has been deleted. *"The payment was declined"* is not exceptional — declined payments are a routine, expected, designed-for part of taking payments.

Using exceptions for these — `throw new CustomerNotFoundException()` when a lookup misses, caught three layers up and turned into a 404 — is *exceptions as flow control*, and it is the original sin from which most exception abuse descends. It is slow (of which more below), it is hard to follow (the control flow leaps across layers invisibly), and it muddies the genuine signal: when *everything* throws, an exception no longer means *"something is wrong"*; it means *"something happened"*, and the operations team learns to ignore the noise.

For expected failure, the right tool is a *return value that represents the failure*. A `Result<T, Error>` (a discriminated union — see [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)). A nullable return for *"not found"*. A `bool TryParse`-style pattern. The failure is *in the signature*, where the caller can see it and the compiler can remind them to handle it — rather than hidden in a `throw` that nothing enforces.

```csharp
// Exceptions as flow control — wrong.
public Customer GetCustomer(CustomerId id)
{
    var customer = _repo.Find(id);
    if (customer is null)
        throw new CustomerNotFoundException(id);   // expected outcome, thrown as exceptional
    return customer;
}

// The failure in the signature — right.
public Customer? FindCustomer(CustomerId id) => _repo.Find(id);
// or, when the caller needs to know WHY:
public Result<Customer, LookupError> FindCustomer(CustomerId id) => /* ... */;
```

## The catch block that should make you ashamed

There is one construct in exception handling that is almost always wrong, and that I have seen in almost every codebase I have ever opened. It is the *empty catch*, or its slightly more respectable cousin, the *catch-and-swallow*.

```csharp
try
{
    await DoSomethingImportant();
}
catch (Exception)
{
    // nothing. or, worse: // TODO: handle this
}
```

This says, in code: *"if anything at all goes wrong here — any exception, of any kind, for any reason — do nothing, tell no one, and carry on as though it succeeded."* It converts a loud, traceable failure into a silent, invisible one. The operation did not happen; nobody knows; the data is now subtly wrong; and the bug report, when it eventually arrives, contains no exception, no stack trace, and no clue, because the one piece of evidence was caught and binned at the scene.

The empty catch is not error handling. It is error *hiding*, and it is responsible for a disproportionate share of the bugs that take days rather than minutes to find. If you take nothing else from this chapter: **never write an empty catch block.** If you genuinely cannot handle an exception, do not catch it — let it propagate to something that can, or to the top, where it will at least be logged.

## `catch (Exception)` is not a strategy

Closely related: catching the base `Exception` type. This catches *everything* — the `OutOfMemoryException` you cannot do anything about, the cancellation exception you must not swallow, the `NullReferenceException` that is a bug you want to know about — all caught by the same indiscriminate net intended for the one specific failure you were actually worried about.

Catch the *specific* exception you can actually handle:

```csharp
try
{
    return await _http.GetAsync(url);
}
catch (HttpRequestException ex)        // the specific thing that can go wrong here
{
    _logger.LogWarning(ex, "Upstream call to {Url} failed", url);
    return FallbackResponse;
}
```

`HttpRequestException` is a thing you can sensibly handle — log it, fall back, retry. `Exception` is not, because it includes a hundred things you cannot. The specific catch documents *what you expected to go wrong and what you decided to do about it*; the broad catch documents only that you gave up trying to be specific.

## Where try/catch genuinely belongs

Having spent several paragraphs on the misuse, let me be constructive about the legitimate homes.

**At system boundaries.** The point where your application calls something it does not control — a database, an HTTP API, the filesystem, a message broker. These genuinely *can* fail in exceptional ways (the network is down), and the boundary is the right place to catch the specific failure, decide on a retry or a fallback, and translate it into the application's own vocabulary.

**At the top of the application.** The global handler — the ASP.NET exception-handling middleware from [Chapter 32](../05_Architecture/32_Middleware.md) — that catches anything that escaped everywhere else, logs it with full context, and turns it into a clean 500 response rather than a leaked stack trace. This is the safety net, and it should be the *only* place that catches broadly, precisely because it is the place whose job is *"something unforeseen got all the way here."*

**Around retryable operations.** A transient-fault policy (Polly and its kind) that catches the specific transient exceptions, waits, and retries. This is exception handling as a deliberate, bounded strategy, not as a panic response.

**In a `finally`, for cleanup that must happen regardless.** Though for the common case — disposing a resource — the `using` statement is the cleaner expression of the same idea.

## Performance, in passing

This is one of the cases where the performance argument reinforces the design argument rather than merely accompanying it.

Throwing and catching an exception is *expensive* — not catastrophically so, but on the order of *thousands* of times more expensive than a simple return, because the runtime has to capture a stack trace, walk the stack looking for a handler, and unwind it. For a genuinely exceptional case that happens rarely, this cost is irrelevant — who cares if the once-a-day database-outage path takes a microsecond longer. But for *exceptions as flow control* — the `CustomerNotFoundException` thrown on every missed lookup, in a loop, thousands of times a request — the cost is real and measurable, and it shows up in profiles as a mysterious slowness that traces back to a `throw` being used where a `return` belonged.

So the expected-failure-as-return-value rule is not only cleaner; it is dramatically faster on the paths where it matters. The fast way and the right way, once again, turn out to be the same way.

> *Using exceptions for ordinary control flow is rather like pulling the fire alarm to announce that the kettle has boiled — effective, in the narrow sense that everybody does indeed stop what they are doing and attend to you, and a habit that ensures nobody will believe you on the day the building is actually alight.*

## The smells

- An empty `catch` block, or one containing only a comment — error hiding, never error handling.
- `catch (Exception)` anywhere except the application's top-level handler — the net is too wide.
- A custom exception type thrown for an expected, ordinary outcome (`NotFound`, `Invalid`, `Declined`) — that wanted a return value.
- A `throw` inside a loop that runs on every iteration of the normal path — exceptions as flow control, paying the cost every time.
- `catch (Exception ex) { throw ex; }` — this rethrow *resets the stack trace*; use a bare `throw;` to preserve it, or do not catch at all.
- A method whose signature gives no hint it can fail, that fails by throwing — the failure should be in the signature.

## Recap

- Exceptions are for the exceptional — the breakdown of assumed conditions, not the expected outcomes.
- Expected failures (not found, invalid, declined) belong in the return type — a `Result`, a nullable, a `Try` pattern — not in a `throw`.
- Never write an empty catch. Never catch `Exception` except at the top-level safety net.
- Legitimate homes for `try/catch`: system boundaries, the global handler, retry policies, and `finally`-for-cleanup (usually a `using`).
- Throwing is thousands of times slower than returning; exceptions-as-flow-control is slow as well as wrong.

## Onwards

The next chapter takes the humble `if` statement and argues that every branch is a small tax on the reader — and that a surprising number of the branches in a typical codebase are avoidable through polymorphism, pattern matching, and a few other techniques for letting the structure of the data, rather than a thicket of conditionals, decide what happens.
