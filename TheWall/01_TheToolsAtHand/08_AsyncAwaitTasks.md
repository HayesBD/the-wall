# Chapter 8 — Async, Await, and the Task Machine

The complaints I hear most often about asynchronous code in C# fall into one of three categories. The first is *"async is hard."* The second is *"async is leaky — once one method is async, all of them have to be."* The third is *"my application deadlocked and I have no idea why."*

All three are true. None of them is async's fault.

Async, in C#, is one of the cleanest implementations of a notoriously difficult problem in any mainstream language. It is also a feature whose surface — two keywords, a return type, a touch of pattern matching — conceals a great deal of compiler-generated machinery, and most of the difficulties people report come, on closer inspection, from misunderstanding that machinery rather than from any inherent awkwardness in the model.

This chapter is an attempt to lift the bonnet just enough to see the engine.

## What `async`/`await` actually compiles to

When you write:

```csharp
public async Task<Customer> LoadCustomerAsync(int id)
{
    var raw = await _db.Customers.FindAsync(id);
    var enriched = await _enrichment.EnrichAsync(raw);
    return enriched;
}
```

…the compiler does not, at runtime, *pause* the method and *resume* it. The compiler rewrites your method into a **state machine** — a hidden class with a switch over an integer field representing where the method currently is, plus storage for any local variables that need to survive across an `await`.

Roughly, what the compiler emits is something like:

1. A state machine struct with a field for each local that crosses an `await`.
2. A `MoveNext()` method containing a switch over the current state.
3. State `0` runs the code up to the first `await`, then schedules a continuation, then returns.
4. When the awaited task completes, the continuation re-enters `MoveNext()` at state `1` — which is the code between the first and second `await`.
5. And so on, until the final state returns a value (or sets an exception) on the `Task<T>` that the original caller received.

The implications of this are quietly profound.

- **Your method does not block.** It surrenders the thread the moment it hits an `await` and resumes on whichever thread is available when the task completes.
- **Locals are preserved by the state machine, not the stack.** This is why captures inside `async` methods sometimes behave subtly differently from synchronous ones.
- **Exceptions are stored on the returned `Task`.** They are not thrown until someone awaits or otherwise observes the task. Hence "unhandled exception in a fire-and-forget task" being a thing.

You do not need to write state machines. You do need to know they are what your `async` method really is.

## What a `Task` actually is

A `Task` is not a thread. A `Task` is *a handle to work that may have completed, may be in progress, or may not have started yet.* It carries a status, a result (for `Task<T>`), an exception (if it faulted), and a list of continuations to run when it completes.

A `Task` may be entirely synchronous internally — `Task.FromResult(42)` is a completed task that did no asynchronous work at all. A `Task` may be backed by genuinely asynchronous I/O — a network read, a file read, a database query — in which case it is the I/O completion port mechanism, not a thread, that signals completion. A `Task` may be backed by a thread-pool work item — `Task.Run(() => ...)` — in which case there is, briefly, a real thread doing real work somewhere on your behalf.

The confusion many new async developers have is to imagine that *every* `Task` corresponds to a thread spun up to do its work. That is the exception, not the rule. The most valuable property of true async I/O is that it is *not* using a thread while it waits — the thread is freed back to the pool, ready to serve another request, and your method is resumed on whichever thread is convenient when the I/O completes.

## The three classic disasters

**`async void`.** Anywhere except event handlers, this is wrong. An `async void` method has no `Task` to return, which means: exceptions thrown inside it have nowhere to go and will crash the process; nothing can await its completion; it is invisible to your test framework's async support. The fix is `async Task` (or `async ValueTask`). If you find an `async void` in a non-event-handler context, treat it as a bug.

**`.Result` and `.Wait()` on a `Task`.** These block the calling thread until the task completes. In a context with a synchronisation context — historically ASP.NET classic, WPF, WinForms — this can and does deadlock. The async machinery wants to resume on the original context. The original context is blocked waiting for the result. Nobody is going anywhere.

ASP.NET Core dispensed with the default synchronisation context, which has made the deadlock less common but no less embarrassing when it appears. The rule remains: in async code, await; in sync code that must call async, use `GetAwaiter().GetResult()` and only at the very edge.

**The forgotten `await`.** A method that returns a `Task` and is called without `await` will run to its first asynchronous suspension and then return a hot, unawaited `Task`. If the task eventually faults, the exception is observed by no one; the operation completes (or does not) without telling anyone; and the bug appears as missing data three weeks later. Modern analysers flag this; treat the warning as an error.

> *Calling `.Result` on a `Task` from a UI thread is the programming equivalent of locking yourself in a room and then phoning a locksmith on your own number to come and let you out.*

## `ConfigureAwait(false)` — the library writer's discipline

By default, an `await` captures the current synchronisation context and resumes there. In a UI app this means *back on the UI thread*, which is usually what you want at the application level. In a library, you almost never need to be back on a specific context, and the cost of capturing one is a small allocation and the *possibility* of contributing to a deadlock in a calling application.

The discipline:

```csharp
// In application code (controllers, view models, page handlers):
var x = await Foo();

// In library code (everything else):
var x = await Foo().ConfigureAwait(false);
```

This is one of those rules that earns its keep silently. Adopt it in libraries and never again ship the deadlock to a customer.

## `ValueTask` — when allocations matter

Every `Task<T>` is, at its heart, a small object on the heap. For hot paths that *usually* complete synchronously — a cache read that hits ninety-eight percent of the time, an internal pipeline stage — the allocation per call adds up.

`ValueTask<T>` is the same idea wrapped in a struct that can hold either a `T` directly (for the synchronous case, no allocation) or a `Task<T>` (for the asynchronous case, just as before). The trade-off: `ValueTask` is more restrictive — you should await it at most once, and you should not block on it — and so it is not the general-purpose tool. Reach for it on hot paths where benchmarks have shown task allocation is meaningful. Leave `Task` as the default everywhere else.

## Performance, in passing

The async/await pattern is, in well-written code, one of the largest single performance wins available to a C# application. Not because individual operations are faster — they are not — but because async I/O *releases the thread* during the wait. A traditional synchronous web application needs roughly one thread per concurrent request. An async one needs roughly one thread per *CPU-bound second* of work, which is often a tiny fraction of that. Memory traffic falls, context switches fall, scaling behaviour transforms.

The cost: each `async` method incurs a small overhead per call (the state machine, the task allocation if reference-typed). For tight inner loops that complete in microseconds, this overhead can dominate. The rule: async at the I/O boundary, sync within tight computational loops, and measure if you are not sure.

## The smells

- `async void` anywhere outside an event handler.
- `.Result` or `.Wait()` anywhere except an explicit synchronous-shim layer.
- `await Task.Run(() => CpuBoundWork())` called from a server endpoint — you have added thread-pool overhead in exchange for nothing.
- A library method whose body is `return SomeAsyncMethod().Result;` — same fault, the deadlock just hasn't found you yet.
- `Task.Run` used as if it were `async` — it schedules work on the thread pool, which is the right move for *CPU-bound* work and the wrong move for I/O.
- Method names that do not end in `Async` despite returning `Task` — a small thing with a large readability cost.

## Recap

- `async`/`await` compiles to a state machine; understanding that explains almost every async surprise.
- A `Task` is a handle to future work, not a thread.
- `async void`, `.Result`, and forgotten `await`s are the three classic disasters.
- `ConfigureAwait(false)` in libraries; default in application code.
- Reach for `ValueTask` only when allocations have been measured and matter.

## Onwards

The next chapter steps from async — which is mostly about giving threads up — to threading proper, which is about putting threads to work. `Task.Run`, `Parallel.ForEach`, channels, locks, and the rare and specific situations in which you actually need to think about a thread by name.
