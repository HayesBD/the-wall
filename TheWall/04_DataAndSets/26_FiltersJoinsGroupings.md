# Chapter 26 — Filters, Joins, Groupings: It Was Sets All Along

The previous chapter promised that every query you have ever written is, in mathematical disguise, a small handful of operations on sets. This chapter pays the promise off. We will take the LINQ vocabulary that you have, until now, probably learned as a list of method names — `Where`, `Select`, `Join`, `GroupBy`, and the rest — and identify each of them as exactly one of the operations from the set-theory chapter. By the end of it, the names should feel less like a vocabulary list to memorise and more like the eight ideas of [Chapter 25](./25_SetTheoryForFiveYearOlds.md) with English labels stuck on.

A small note on terminology. LINQ operates on `IEnumerable<T>` (and `IQueryable<T>`), which are *sequences* rather than sets — they may contain duplicates and they have an order. For most of what follows, the distinction is unimportant; where it matters, I will point it out.

## `Where` is filter

The simplest one first. `Where` produces a new sequence containing only the elements of the original that satisfy a condition.

```csharp
var spanielsStartingWithB = spaniels.Where(s => s.Name.StartsWith("B"));
```

In set theory, this is *filter*. *"Give me only the members that satisfy this condition."* The result is a *subset* of the original. Nothing else has changed — same element type, same ordering — just fewer elements.

The SQL equivalent is `WHERE`. The Python equivalent is the comprehension's `if` clause: `[s for s in spaniels if s.name.startswith("B")]`. Every language has a name for this operation, because every language needs one.

## `Select` is map

`Select` produces a new sequence by applying a function to each element of the original. The resulting sequence has the same length as the original but a (possibly) different element type.

```csharp
var names = spaniels.Select(s => s.Name);               // IEnumerable<string>
var totals = orders.Select(o => new { o.Id, o.Total });  // anonymous projection
```

In set theory, this is *map*, or *projection*. The first form transforms each `Spaniel` into a `string`. The second transforms each `Order` into a small anonymous record. The shape changes; the count stays.

The SQL equivalent is the `SELECT` clause itself — *"give me these columns, derived from these rows."* The connection is so direct that one of the LINQ designers, I am told, lobbied for `Select` to be called `Project`. Tradition won.

## `SelectMany` is flatten-then-map (or map-then-flatten)

`SelectMany` is the operation that, in functional-programming circles, is called `flatMap` or *bind*. For each element of a sequence, it produces a sequence of derived elements, and then concatenates all the resulting sequences into a single flat sequence.

```csharp
var allOrderItems = customers.SelectMany(c => c.Orders.SelectMany(o => o.Items));
```

In set theory, this is the *generalised union* — take each customer, derive a sequence of items, and union all the resulting sequences. The output is the flat set of all items across all customers.

The SQL equivalent is the natural cross-table flatten that a `JOIN` produces, viewed from the perspective of the parent table. The Python equivalent is a comprehension with two `for` clauses: `[item for c in customers for o in c.orders for item in o.items]`.

`SelectMany` is one of the most powerful operations LINQ provides and one of the most underused. Reach for it whenever you have a sequence of things that themselves contain sequences and you want a single flat sequence at the end.

## `Join` is filtered Cartesian product

This is the operation that justifies the entire set-theoretic detour. A `Join` takes two sequences, conceptually produces every possible pair of elements (the Cartesian product), and then keeps only the pairs whose key fields match.

```csharp
var customerOrders = customers.Join(
    orders,
    customer => customer.Id,
    order => order.CustomerId,
    (customer, order) => new { customer.Name, order.Total });
```

In set theory, this is `{(c, o) : c ∈ customers, o ∈ orders, c.Id = o.CustomerId}` — every customer-order pair from the Cartesian product, filtered to those whose IDs match. The framework (LINQ in memory, the database engine for `IQueryable`) is then *extremely* clever about not actually materialising the full Cartesian product first; it uses hash tables, sort-merge joins, and index lookups to get to the same answer cheaply. But the conceptual shape is *Cartesian-product-then-filter*, and understanding it that way tells you why an unindexed join is slow and an indexed one is fast.

The SQL equivalent is the explicit `INNER JOIN ... ON`. The two are, in any reasonable sense, the same operation written in two different dialects.

## `GroupJoin` is the *one-to-many* variant of join

A `GroupJoin` produces, for each element on the left side, *a sequence* of matching elements from the right side. The result is a one-to-many shape, naturally suited to nested rendering.

```csharp
var customersWithTheirOrders = customers.GroupJoin(
    orders,
    customer => customer.Id,
    order => order.CustomerId,
    (customer, customerOrders) => new { customer.Name, Orders = customerOrders });
```

In set theory, this is *map followed by filter*, where the map produces, for each left element, the subset of right elements that match. It is what you reach for when you want the parent records each carrying their own collection of children.

SQL has no direct equivalent because SQL's relational result is flat; the nesting in `GroupJoin` is something LINQ does in the application layer.

## `GroupBy` is partition

`GroupBy` takes a sequence and splits it into smaller sequences (groups) that share a common key value.

```csharp
var ordersByCustomer = orders.GroupBy(o => o.CustomerId);
// Each group is a sequence of orders sharing a CustomerId.
```

In set theory, this is *partition* — every element of the original lands in exactly one group, no group is empty, and every original element appears in exactly one group. The groups can then be reduced (summed, counted, averaged) to produce one summary per group.

The SQL equivalent is, mercifully, `GROUP BY`. The connection is so direct that, for `IQueryable<T>` against an EF Core source, the LINQ `GroupBy` translates almost literally into a `GROUP BY` clause sent to the database.

## `Distinct` is the recognition that sets don't have duplicates

`Distinct` removes duplicates from a sequence.

```csharp
var uniqueRegions = orders.Select(o => o.Region).Distinct();
```

In set theory, this is the conversion *from a multiset (or sequence) to a set*. Sequences may have duplicates; sets, by definition, do not. `Distinct` is the operation that bridges the gap.

The SQL equivalent is `SELECT DISTINCT`, with the same caveat — and the same trap — that distinguishing duplicates requires the database to compare every pair of resulting rows, which without indexing can be expensive on large result sets.

## `Union`, `Intersect`, `Except` — the three set operations directly

LINQ exposes three operations from the set-theory chapter under their actual mathematical names:

```csharp
var combined = setA.Union(setB);        // ∪
var common   = setA.Intersect(setB);    // ∩
var only     = setA.Except(setB);       // \
```

`Union` is union, with duplicates removed. `Intersect` is intersection. `Except` is set difference. The SQL equivalents are, again, identically named: `UNION`, `INTERSECT`, `EXCEPT`.

Note the distinction between `Union` (which deduplicates) and `Concat` (which simply appends, allowing duplicates). `Union` is the set operation; `Concat` is the sequence operation. In SQL these are `UNION` and `UNION ALL`, with the same distinction.

## `All`, `Any`, `Contains` — the predicates

Three operations that, instead of producing a derived sequence, produce a boolean.

```csharp
var allPaid    = orders.All(o => o.Status == OrderStatus.Paid);  // is every member ...?
var anyOverdue = orders.Any(o => o.IsOverdue);                    // is there a member ...?
var has42      = ids.Contains(42);                                // is this element a member?
```

In set theory, these are *universal quantification*, *existential quantification*, and *membership testing*. They correspond exactly to *"is the filtered subset equal to the original?"*, *"is the filtered subset non-empty?"*, and *"is this element in the set?"*.

The SQL equivalents are `WHERE NOT EXISTS (...)` patterns and the bare `IN` clause. `Any` and `EXISTS` are the same idea wearing different clothes.

## `Aggregate` — the fold

The most general operation, sometimes called *fold* or *reduce* in other languages. Combine all elements of a sequence into a single result by repeatedly applying a function.

```csharp
var total = orders.Aggregate(0m, (running, order) => running + order.Total);
```

In set theory, this is not strictly a set operation — it depends on order, and order is not a thing sets have — but it is the substrate from which `Sum`, `Count`, `Min`, `Max`, `Average`, and the other aggregations are built. Each of those is `Aggregate` with a specific accumulator function pre-supplied.

The SQL equivalents are the aggregate functions: `SUM`, `COUNT`, `MIN`, `MAX`, `AVG`. They appear, naturally, alongside `GROUP BY`, where the partitioning has produced the groups and the aggregate operates within each.

## The full table

| LINQ method | Set-theory name | SQL equivalent |
|---|---|---|
| `Where` | Filter | `WHERE` |
| `Select` | Map / projection | `SELECT <cols>` |
| `SelectMany` | Generalised union | `JOIN` (one-side flatten) |
| `Join` | Filtered Cartesian product | `INNER JOIN ... ON` |
| `GroupJoin` | One-to-many subset | (application-side nesting) |
| `GroupBy` | Partition | `GROUP BY` |
| `Distinct` | Sequence → set | `SELECT DISTINCT` |
| `Union` | Union | `UNION` |
| `Intersect` | Intersection | `INTERSECT` |
| `Except` | Difference | `EXCEPT` |
| `Concat` | Multiset union | `UNION ALL` |
| `All` | Universal quantifier | `NOT EXISTS (... fail)` |
| `Any` | Existential quantifier | `EXISTS` |
| `Contains` | Membership | `IN` |
| `Aggregate` | Fold | `SUM`/`COUNT`/`MIN`/`MAX`/`AVG` |

Fifteen rows. Fifteen ideas. Once internalised, the entire data-querying surface area of C# (and of every other modern language) reveals itself as the same handful of set-theoretic operations rendered in different syntax.

## What this is good for

Beyond the satisfaction of recognising the mathematical structure of code you have been writing for years, knowing the set-theoretic shape of your queries gives you three concrete advantages.

**You can predict performance.** A query whose conceptual shape is *filter-then-map* is O(n). A query whose conceptual shape is *Cartesian-product-then-filter* is O(n × m) without an index, and roughly O(n + m) with one. Knowing which shape you have written tells you, before you run it, what to expect.

**You can refactor safely.** A `.Where(...).Select(...)` and a `.Select(...).Where(...)` produce the same result if the `Where` does not depend on the projected shape — *filter and map commute under certain conditions*, and recognising those conditions lets you reorder for clarity or for performance. The same goes for combining `.Where`s into one and splitting one into many.

**You can write queries that translate.** EF Core's `IQueryable<T>` is, at heart, an attempt to translate LINQ method chains into SQL. Some chains translate cleanly; some do not. The ones that do not are almost always ones where the LINQ shape does not correspond to a single set-theoretic operation — the offending method is doing something procedural that the database engine cannot recover the set-theoretic intent of. Writing in set-theoretic shapes is, *as a happy by-product*, writing in shapes that the database can understand.

> *Writing LINQ without seeing the set-theory underneath is rather like driving a car without quite knowing that the steering wheel turns the front wheels: it works most of the time, and it does not work in the way you expect on the day it really matters.*

## Performance, in passing

A small concrete example to close the loop. Compare these two queries:

```csharp
// Query A
var result = customers
    .Where(c => c.Region == "EU")
    .SelectMany(c => c.Orders)
    .Where(o => o.Total > 1000m);

// Query B
var result = customers
    .SelectMany(c => c.Orders)
    .Where(o => o.Total > 1000m)
    .Where(o => o.Customer.Region == "EU");
```

The two produce the same result. Query A applies the customer-region filter first, then flattens to orders, then applies the total filter. Query B flattens first, then applies both filters. In a hand-iterated `IEnumerable<T>` on a list of a million customers, Query A is dramatically faster — the customer-region filter rejects most customers cheaply, before the order flattening fires.

For `IQueryable<T>` against EF Core, both queries translate to similar SQL and the database optimiser sorts it out. For `IEnumerable<T>` against in-memory data, the order matters very much. *Seeing the set-theoretic shape lets you choose.*

## Recap

- Every LINQ method is exactly one of the operations from set theory.
- `Where` is filter, `Select` is map, `Join` is filtered Cartesian product, `GroupBy` is partition.
- The set operations (`Union`, `Intersect`, `Except`) appear in LINQ under their actual names.
- Seeing the shape of a query lets you predict its cost, refactor it safely, and write versions the database can translate.
- The entire data-querying surface area is fifteen ideas in clean correspondence.

## Onwards

The next chapter takes the set-theoretic vocabulary and applies it to the most important practical question in business software: *how to write LINQ that lands in EF Core without producing the kind of query a DBA can hear from three offices away*. Deferred execution, the great `IQueryable<T>` / `IEnumerable<T>` schism, the N+1 problem, projection before materialisation, and the small rituals that turn a four-second page load into a forty-millisecond one.
