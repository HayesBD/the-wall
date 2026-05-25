# Chapter 9 — Threading: When the Task Isn't Enough

The previous chapter spent its time on giving threads *up* — which is, statistically, what most modern applications, most of the time, should be doing. This chapter is about putting threads *to work*, which is a smaller, more specialised, and considerably more dangerous business.

The good news is that the situations where you genuinely need to think about threads are rarer than the average developer imagines. The bad news is that, when you are in one of them, the .NET concurrency machinery offers a slightly intimidating buffet of options — `Task.Run`, `Parallel.ForEach`, `Parallel.ForEachAsync`, `Channel<T>`, `Thread`, `lock`, `SemaphoreSlim`, `Interlocked`, `Barrier`, `ConcurrentDictionary`, and so on — and choosing the wrong one is a fast way to ruin both your performance and your weekend.

## Concurrency and parallelism — not the same thing

A distinction worth making clearly before we proceed, because it underlies almost every decision in this chapter.

**Concurrency** is about *structure* — your program is dealing with several things at once. Whether or not those things are running at the same physical instant is irrelevant. A single-threaded JavaScript program serving thousands of HTTP requests via an event loop is concurrent. An old-school cooperative-multitasking operating system was concurrent.

**Parallelism** is about *execution* — your program is using multiple CPU cores to do several things at the same physical instant. This requires multiple threads, multiple cores, and work that can be meaningfully split.

Most web applications are concurrency problems wearing parallelism's clothing. They serve many requests at once, but each individual request is mostly waiting on I/O — a database, a downstream service, a cache. Async I/O gives you concurrency without parallelism, and that is usually what you wanted.

Parallelism comes in only when you have **CPU-bound work** that can be split into independent pieces — image processing, financial simulation, batch numerical work. For those, threads earn their keep.

## The thread pool

Almost all the threading machinery in modern .NET — `Task.Run`, `Parallel.*`, `await` continuations, timer callbacks — runs on the **thread pool**. The thread pool is a small standing army of worker threads, managed by the runtime, sized roughly to the number of CPU cores plus a margin for blocking work.

You should not, with rare and specific exceptions, create your own threads. The thread pool already exists. It is faster to schedule work onto it than to spawn a new thread. It also has the runtime's full attention when it comes to blocking-work detection and pool growth.

The exception — and we will come to it shortly — is **long-running, dedicated** work: a background service, a custom scheduler, a UI thread. For those, a `Thread` (or, more commonly, a `Task` created with `TaskCreationOptions.LongRunning`) is the right answer.

## `Task.Run` — the right and wrong reaches

`Task.Run(Func<T>)` schedules its argument on the thread pool and returns a `Task<T>` that completes when the work does. It is the simplest possible way to say *"do this on another thread."*

It is also, regularly, the wrong reach.

**Wrong**: calling `Task.Run` from a server endpoint to wrap an already-`await`able call.

```csharp
// Bad — pointless thread hop.
public async Task<Customer?> GetAsync(int id) =>
    await Task.Run(async () => await _db.Customers.FindAsync(id));
```

`FindAsync` is already async. Wrapping it in `Task.Run` does not make it more parallel; it just adds a thread-pool work item, a context switch, and a small amount of allocation. The request thread is freed for the duration of the I/O regardless. The `Task.Run` is theatre.

**Right**: calling `Task.Run` from a UI thread to push CPU-bound work off the UI thread.

```csharp
// Good — keeps the UI responsive while the work runs.
public async Task RenderAsync()
{
    var bitmap = await Task.Run(() => RenderMandelbrot(1920, 1080, 1000));
    DisplayBitmap(bitmap);
}
```

The CPU work is real; the UI thread is something you absolutely must not block; the thread pool is where the work goes. Here `Task.Run` is the right tool.

**Also right**: parallelising CPU-bound work in a server context where you have headroom.

```csharp
var results = await Task.WhenAll(
    inputs.Select(input => Task.Run(() => ExpensiveCompute(input)))
);
```

But notice: this only helps because `ExpensiveCompute` is CPU-bound and you have multiple cores to give it. If `ExpensiveCompute` were itself I/O-bound, `Task.WhenAll` over the bare async methods would do the same thing without the thread hops.

## `Parallel.ForEach` and `Parallel.ForEachAsync`

When you have a collection of CPU-bound work items, `Parallel.ForEach` is purpose-built:

```csharp
Parallel.ForEach(images, image =>
{
    var thumbnail = GenerateThumbnail(image);
    SaveThumbnail(thumbnail);
});
```

The runtime partitions the collection across thread-pool workers, runs them in parallel, and returns when the last one is done. There is a degree-of-parallelism knob, a cancellation token option, and exception aggregation built in.

For asynchronous workloads — *"download these hundred files in parallel, no more than ten at a time"* — the modern answer is `Parallel.ForEachAsync`:

```csharp
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var bytes = await _http.GetByteArrayAsync(url, ct);
        await File.WriteAllBytesAsync(LocalPath(url), bytes, ct);
    });
```

This is one of the most underused tools in modern .NET. It replaces the entire pattern of *"build a list of tasks, throttle them through a `SemaphoreSlim`, await `Task.WhenAll`"* with a single, well-tested API.

## `Channel<T>` — producer/consumer done properly

When you have *one or more producers* generating work and *one or more consumers* processing it, and the two need to run independently, the right tool is `Channel<T>` from `System.Threading.Channels`.

```csharp
var channel = Channel.CreateBounded<Order>(capacity: 100);

// Producer
_ = Task.Run(async () =>
{
    await foreach (var order in ReadIncomingOrdersAsync())
        await channel.Writer.WriteAsync(order);
    channel.Writer.Complete();
});

// Consumer
await foreach (var order in channel.Reader.ReadAllAsync())
    await ProcessAsync(order);
```

Channels handle back-pressure (the bounded variant blocks the producer when the buffer is full), completion signalling, and the producer/consumer dance without you having to write any of it yourself. They are the modern replacement for the older `BlockingCollection<T>`, and they integrate with async naturally.

## Locks — the bare minimum

When two threads need to mutate the same data, you need synchronisation. The simplest mechanism is the `lock` keyword:

```csharp
private readonly object _gate = new();
private int _count;

public void Increment()
{
    lock (_gate)
    {
        _count++;
    }
}
```

A few small disciplines prevent most of the pain:

- **Lock on a `private readonly object`**, never on `this`, a type, or a string. The first because external code can lock on the same `this` and deadlock you; the latter two because they are shared across the whole program.
- **Hold the lock for the shortest time possible.** Compute outside the lock; mutate inside it.
- **Never `await` inside a `lock`.** The compiler will not let you, which is the language saving you from yourself.
- **For single-integer arithmetic, prefer `Interlocked`.** `Interlocked.Increment(ref _count)` is lock-free, atomic, and considerably faster.
- **For async-aware mutual exclusion, use `SemaphoreSlim`** with `await semaphore.WaitAsync()`. It is the only async-friendly lock the BCL provides.

For shared collections, reach for the `Concurrent*` family — `ConcurrentDictionary<TKey, TValue>`, `ConcurrentQueue<T>`, `ConcurrentBag<T>` — before reaching for a `lock` around an ordinary collection. The implementations are subtle, well-tested, and faster than the equivalent hand-rolled lock-and-`Dictionary` would be.

> *Sharing mutable state across threads without a plan is the programming equivalent of putting two strangers in a hire car and asking them both to drive: there is, briefly, a sense of progress, followed by a great deal of expensive paperwork.*

## When you actually need to think about a thread

The cases where a bare `Thread` is the right answer are rare and specific.

- **Long-running background services.** A `BackgroundService` (or a `Task` with `TaskCreationOptions.LongRunning`) that runs for the lifetime of the process — a poller, a queue reader, a dedicated worker. Do not put these on the thread pool, because they will hold a pool thread permanently and starve everything else.
- **UI threads.** WPF, WinForms, and MAUI have dedicated UI threads, and the framework will be cross with you if you touch UI from anywhere else. Use `Dispatcher.Invoke` or the equivalent to marshal back.
- **Native-interop callbacks that demand a specific thread affinity.** Some COM components, some old graphics APIs. If you do not know whether you are in one of these cases, you are not.

For everything else: thread pool.

## Performance, in passing

Threads are not free. Each thread costs around a megabyte of stack space and a meaningful chunk of OS bookkeeping. Spinning up a thread per request, which the early Java servlet model encouraged, scales to perhaps a thousand concurrent connections before the OS itself becomes the bottleneck. Async I/O on a small thread pool scales to tens or hundreds of thousands.

Parallelism, where it applies, gives near-linear speedup for embarrassingly parallel CPU work, sub-linear for work with any sharing or contention, and *negative* speedup if the work is actually I/O-bound and you have introduced thread overhead for no gain. Measure before celebrating.

## The smells

- `new Thread(...)` outside the rare cases above — you wanted the thread pool.
- `Task.Run` wrapping an already-async call — you added overhead for nothing.
- A `lock (this)` — you have invited external code to deadlock you.
- A `lock` held across an HTTP call, a database query, or any other I/O — every other request that wants the same lock is now queued behind it.
- A `Dictionary<TKey, TValue>` accessed from multiple threads with a `lock` wrapped around every read and write — you wanted a `ConcurrentDictionary`.
- `Thread.Sleep` anywhere in production code — you wanted `await Task.Delay`.

## Recap

- Concurrency is about structure; parallelism is about execution. Most server work needs the former, not the latter.
- The thread pool is the default; reach for `Thread` only for long-running, dedicated work or UI affinity.
- `Task.Run` is for pushing CPU-bound work off the calling thread, not for wrapping async I/O.
- `Parallel.ForEachAsync` and `Channel<T>` are the modern, well-engineered tools for parallel async work and producer/consumer pipelines respectively.
- Locks belong on private objects, around the shortest possible critical sections, and never with `await` inside.

## Onwards

That closes Part I — the tools at hand. From here we change altitude. Part II turns to the most cherished pattern in mainstream object-oriented software — the model — and gives it a long, careful, fair-minded examination. It will hold up better than some of its detractors claim. It will hold up less well than some of its defenders insist. Let us begin.
