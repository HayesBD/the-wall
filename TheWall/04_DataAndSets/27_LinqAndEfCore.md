# Chapter 27 — LINQ and EF Core: Writing Queries That Don't Make DBAs Cry

The previous chapter established that LINQ is set theory wearing different clothes. This chapter is about what happens when those clothes are draped over EF Core, and the small disciplines that mean the difference between a query the database can answer cheaply and a query that, in the gentle phrasing favoured by experienced DBAs, *"warrants a conversation."*

I want to be clear about what this chapter is and is not. It is not a tutorial on EF Core's API surface — the official documentation is excellent and I have no interest in restating it. It is a chapter about the *handful of recurring shapes* that, in my experience, separate well-performing EF Core code from the kind that makes the operations team think wistfully about an earlier career.

We will assume you know how to register a `DbContext`, define an entity, and write a basic query. From there.

## The `IQueryable<T>` / `IEnumerable<T>` schism

The single most important concept in EF Core, and the one most commonly misunderstood. The same LINQ method names — `Where`, `Select`, `OrderBy` — exist on both interfaces, and they look interchangeable. They are not.

**`IEnumerable<T>`** is *"a sequence of objects in memory."* Operations on it are evaluated by walking the sequence in C#. A `Where` clause is a C# predicate executed in your process, against objects already loaded from somewhere.

**`IQueryable<T>`** is *"a sequence whose definition we have not yet evaluated."* Operations on it build up an expression tree that, when finally materialised, is translated into a SQL query and sent to the database. The `Where` clause becomes a SQL `WHERE`; the `Select` becomes a `SELECT` list.

The schism arises at the exact moment your query *stops being* `IQueryable<T>` and *starts being* `IEnumerable<T>` — and the moment is often invisible.

```csharp
// IQueryable — the whole thing translates to one SQL query.
var result = _db.Customers
    .Where(c => c.Region == "EU")
    .Where(c => c.LifetimeSpend > 1000m)
    .Select(c => new { c.Id, c.Name })
    .ToList();

// IQueryable, then IEnumerable — the .ToList() pulls everything,
// then the .Where runs in memory on the full materialised set.
var result = _db.Customers
    .ToList()
    .Where(c => c.Region == "EU");
```

The second example *works* — it produces the correct result. It also loads every customer in the database into memory and then filters. For a hundred customers, this is unmeasurable. For a hundred million, it is the kind of mistake that gets a paragraph in the post-mortem.

The rule: *keep your queries as `IQueryable<T>` until you genuinely want the result*. Every `.ToList()`, `.ToArray()`, `.AsEnumerable()`, and `.ToDictionary()` is a materialisation boundary. Anything chained after one runs in C#, not in the database.

## Deferred execution

A close cousin. `IQueryable<T>` (and `IEnumerable<T>` from `yield return`-shaped methods) does not, by itself, *do anything*. It is a *recipe* for a query, not a query. The recipe is executed only when something materialises it — a `ToList()`, a `foreach`, a `Count()`, a `First()`, a `Sum()`.

```csharp
var query = _db.Orders.Where(o => o.Total > 1000m);
// Nothing has run yet.

var list = query.ToList();
// Now the SQL has been sent and the results materialised.

var count = query.Count();
// And now it has run AGAIN, as a different SQL query (SELECT COUNT(*)).
```

The fact that `query` can be enumerated *multiple times*, each time hitting the database afresh, is sometimes useful and sometimes a horror. The horror version: a `foreach` iterates an `IQueryable<T>` directly, and inside the loop body something else iterates the same `IQueryable<T>` again. The database is now serving two parallel queries for the same data.

The rule: *if you intend to use a result more than once, materialise it once and reuse the materialised collection*.

## The N+1 problem and its four shapes

The most common single performance problem in EF Core applications, and one that has been understood for the best part of two decades. The pattern is always the same: a query returns N rows, and for each row, *another* query is fired to fetch related data. The total is N+1 queries where one would have done.

The four shapes I see most often.

**Shape one — the bare loop.**

```csharp
foreach (var customer in _db.Customers.ToList())
{
    Console.WriteLine($"{customer.Name}: {customer.Orders.Count}");
}
```

`customer.Orders` is a navigation property. With lazy loading enabled, accessing it fires a query. With lazy loading disabled, accessing it returns an empty collection — which is its own bug. Either way, this is *one query per customer*, plus the one to load the customers in the first place.

The fix: `Include`.

```csharp
foreach (var customer in _db.Customers.Include(c => c.Orders).ToList())
    /* ... */
```

One query with a join, instead of N+1.

**Shape two — the projection that walks a navigation.**

```csharp
var summaries = _db.Customers
    .Where(c => c.Region == "EU")
    .ToList()
    .Select(c => new {
        c.Name,
        OrderCount = c.Orders.Count
    });
```

The `ToList()` materialised the customers; the subsequent `.Select` runs in memory; the `c.Orders.Count` then triggers lazy loading per customer. The fix is to push the projection *before* materialisation, so the database does the counting:

```csharp
var summaries = _db.Customers
    .Where(c => c.Region == "EU")
    .Select(c => new {
        c.Name,
        OrderCount = c.Orders.Count()   // translates to a SQL subquery
    })
    .ToList();
```

One query. The database aggregates as part of the same statement. The change is small; the performance difference is sometimes orders of magnitude.

**Shape three — the constructor that loads.**

```csharp
public class OrderSummary
{
    public OrderSummary(Order order)
    {
        Number = order.Number;
        CustomerName = order.Customer.Name;   // triggers lazy load
    }
}

var summaries = orders.Select(o => new OrderSummary(o)).ToList();
```

The same pattern as shape two, hidden inside a constructor. The cure is the same: project the *fields*, not the *entity*.

**Shape four — the await-inside-foreach.**

```csharp
foreach (var customer in customers)
{
    var orders = await _db.Orders
        .Where(o => o.CustomerId == customer.Id)
        .ToListAsync();
    /* ... */
}
```

Each iteration of the loop awaits its own query. The cure: load all the related orders in one query (`Where(o => customerIds.Contains(o.CustomerId))`) before the loop, then group them in memory.

## Projection before materialisation

A rule worth stating on its own line because it is the single highest-impact discipline in EF Core code.

**Project to the shape you need before you call `ToList()`. Never after.**

```csharp
// Bad — materialises every column of every customer, even though we only need two.
var names = _db.Customers.ToList().Select(c => new { c.Id, c.Name });

// Good — the SELECT clause asks for only the two columns we need.
var names = _db.Customers.Select(c => new { c.Id, c.Name }).ToList();
```

The difference is one position in a method chain. The performance difference is the difference between *"transfer every byte of every customer row from the database to the application"* and *"transfer two columns, with only the bytes those columns occupy."* For wide tables — and most real tables are wider than people remember — the saving is substantial.

This is also the lever that turns the model-bloat problem from [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md) from a structural problem into a query-shape problem. A god entity with thirty columns is, projected properly at the query, six columns of bytes over the wire. The structure is still wrong; the *operational* damage is reduced.

## Tracking and no-tracking

EF Core tracks every entity it materialises, by default, so that subsequent mutations to those entities can be detected and persisted on `SaveChanges`. For *write* operations, this is the right behaviour. For *read* operations — anything that produces data for display, for reporting, for export — the tracking is overhead with no benefit.

```csharp
// Default — tracked. Use for entities you intend to modify.
var customer = await _db.Customers.SingleAsync(c => c.PublicId == id);

// No tracking — faster, lower memory. Use for read-only paths.
var customer = await _db.Customers
    .AsNoTracking()
    .SingleAsync(c => c.PublicId == id);
```

A common discipline: configure your read-mostly contexts to be no-tracking by default (`UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)`), and opt back in to tracking explicitly when you actually need it.

For projections that produce non-entity types (`Select(c => new { c.Id, c.Name })`), tracking is automatically off — there is nothing to track. The choice only matters when you are pulling whole entities.

## Split queries

When a single query joins multiple `Include`d collections, the resulting SQL becomes a Cartesian product across the joined tables — and the wire traffic balloons accordingly.

```csharp
var customers = await _db.Customers
    .Include(c => c.Orders)
    .Include(c => c.Addresses)
    .ToListAsync();
```

A customer with five orders and three addresses produces fifteen rows over the wire, each carrying the customer's columns repeated. EF Core deduplicates in memory, but the database is sending the full Cartesian product.

For deep includes or wide entities, *split queries* — one query per collection — are usually cheaper:

```csharp
var customers = await _db.Customers
    .AsSplitQuery()
    .Include(c => c.Orders)
    .Include(c => c.Addresses)
    .ToListAsync();
```

Three round-trips, none with the wire amplification. The trade-off: an extra round-trip is wall-clock latency, and for high-latency database connections this is not always a win. Measure if you are not sure. As a default, use single queries for one collection, split queries for two or more.

## Compiled queries

For queries that are executed *very* frequently — the per-request lookup of the current user, the cached-page hot path — the cost of EF Core's expression-tree compilation, even with its caching, becomes measurable. The remedy is the compiled query:

```csharp
private static readonly Func<AppDbContext, Guid, Task<Customer?>> _byPublicId =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Customers.SingleOrDefault(c => c.PublicId == id));

public Task<Customer?> GetAsync(Guid publicId) => _byPublicId(_db, publicId);
```

Compiled once, executed many times, with the per-call overhead reduced to roughly the cost of executing the SQL itself. Reach for this when a single query is identified as a hot path in profiling. Do not reach for it everywhere; the ceremony is real.

## What I will not pretend

EF Core is, in my view, the best ORM in the .NET ecosystem and one of the best in any ecosystem. It is also, on hot paths or complex queries, sometimes the wrong tool. There are workloads — bulk imports, very complex reports, streaming exports, intricate dynamic queries — where a thin wrapper over raw SQL (Dapper) or, indeed, the bare ADO.NET API is the right reach.

The trick is knowing which workload you have. The 80% case is well-served by EF Core with the disciplines above. The 20% case wants a different tool, and recognising the boundary is part of the craft.

> *Refusing to use raw SQL anywhere in an application "because we have an ORM" is rather like refusing to walk anywhere "because we have a car" — a perfectly defensible policy when the journey is half a mile and you have luggage; a slightly absurd one when the journey is to the end of the driveway to fetch the post.*

## Performance, in passing

Numbers, again, are representative rather than universal. For a typical CRUD endpoint reading a customer and three related collections:

- A naive `Include` chain returning whole entities, with default tracking, on a wide table: roughly 200ms.
- The same query with `AsNoTracking` and projection to a slim DTO: roughly 25ms.
- The same query as a compiled query with the same projection: roughly 8ms.

The first to the second is an eight-fold improvement for one line of code (`AsNoTracking`) and one shape change (project to DTO). The second to the third is another threefold for promoting the query to compiled form. None of these is the result of database optimisation; all of them are application-side discipline.

## The smells

- A `.ToList()` followed by `.Where(...)` — the filter is running in memory after a full table scan.
- A `.Include(...)` returning whole entities when only two columns are read at the call site — project earlier.
- A `foreach` over an `IQueryable<T>` that fires the query multiple times — materialise once, reuse.
- A read endpoint without `AsNoTracking()` — the change tracker is overhead for nothing.
- A `.Count()` followed shortly by a `.ToList()` of the same query — two database round-trips for one logical operation.
- A constructor that takes an entity and immediately accesses a navigation property — guaranteed N+1 unless the caller is very careful.
- A `Skip(N).Take(M)` on a query without an `OrderBy` — the database is free to return rows in any order, and your pagination will, occasionally, repeat rows or skip them.

## Recap

- `IQueryable<T>` runs in the database; `IEnumerable<T>` runs in C#. Know which one you are holding.
- Project to the shape you need *before* materialising. This is the single largest performance discipline in EF Core code.
- The N+1 problem comes in four common shapes. All four have direct cures.
- `AsNoTracking()` for read paths, tracking for write paths. The default in most apps is wrong by laziness rather than by design.
- Split queries when multiple collections are included. Compiled queries when a single query is in a hot path.
- EF Core is the right tool for 80% of the work; recognise the 20% where raw SQL belongs.

## Onwards

The next chapter steps from *querying* the data to *shaping* it — the relationships between entities and the small modelling choices (one-to-many, many-to-many, optional vs required) that, badly chosen, become the structural mistakes whose long shadows reach back through every chapter of this Part.
