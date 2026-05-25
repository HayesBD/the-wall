# Chapter 23 — Collections: What Most People Get Wrong

C# has, in 2026, somewhere between fifteen and twenty distinct collection types you might reasonably use in production code, depending on how you count. Most working developers reach for two of them — `List<T>` and `Dictionary<TKey, TValue>` — for very nearly everything. This is, mostly, defensible. It is also, often, the wrong reach.

The trouble with the two-tool habit is not that those two tools are bad. They are not; they are excellent tools, and the engineering put into their internals deserves more respect than it usually gets. The trouble is that the cost of choosing wrong is invisible until your codebase is big enough that the wrong choice has been made a few hundred times, at which point the cumulative slowness, allocation pressure, and accidental data-shape ambiguity become a real and difficult-to-isolate problem.

This chapter is a tour of the collection types, with the working rule for when each is the right reach.

## The core three

**`T[]`** — the humble array. Fixed length. Contiguous. The fastest collection by every meaningful measure. No allocation per add (there are no adds). Use this when you know the length in advance and it will not change.

```csharp
int[] daysOfWeek = new int[7];
string[] vowels = ["a", "e", "i", "o", "u"];
```

The collection-expression syntax (`[a, b, c]`) makes arrays as ergonomic to construct as lists, with none of the per-add allocation cost. For a fixed set, this is the right tool.

**`List<T>`** — the dynamic array. Backed by a `T[]` internally; grows by doubling its capacity when full. Excellent for unknown-but-bounded lengths and for the common case of *"build up a collection then iterate it."*

```csharp
var items = new List<Order>();
foreach (var order in source) items.Add(order);
```

Each `Add` is O(1) amortised. Each access by index is O(1) genuinely. Iteration is contiguous. This is the right default for the *"I am building this up dynamically"* case.

**`Dictionary<TKey, TValue>`** — the hash map. O(1) average lookup, insert, and delete. Use this when the access pattern is *"give me the value for this key"* and the key is not numeric and dense.

```csharp
var byEmail = new Dictionary<string, Customer>();
foreach (var c in customers) byEmail[c.Email] = c;
```

The default for keyed lookup. Three small disciplines that prevent most of its pain: choose your key type carefully (records and `readonly record struct`s make excellent keys; mutable classes do not), use `TryGetValue` rather than `ContainsKey` followed by `[]` (one hash, not two), and initialise with a capacity hint when you know the approximate size.

## The next four

**`HashSet<T>`** — the hash set. Membership testing in O(1) average. Use this when the question is *"is this thing in the collection?"* and the collection is more than about twenty items long.

```csharp
var allowedRegions = new HashSet<string>(StringComparer.OrdinalIgnoreCase) { "EU", "UK", "US" };
if (allowedRegions.Contains(request.Region)) /* ... */;
```

The single most common collection mis-choice I see in code reviews is `List<T>.Contains` over a list of a few hundred items, which is O(n) per call. `HashSet<T>.Contains` is O(1). For tight inner loops, this is sometimes a hundredfold speedup with no other change.

**`SortedDictionary<TKey, TValue>` / `SortedSet<T>`** — when ordered iteration matters. O(log n) operations, backed by a tree. Use these when you need *"give me the values in key order"* without sorting at every read.

**`Queue<T>` / `Stack<T>`** — FIFO and LIFO respectively. O(1) operations. Use these when the access pattern is *"add to one end, remove from the other (or the same end)."* If you find yourself using `List<T>.RemoveAt(0)`, you wanted a `Queue<T>` — the `List<T>` version is O(n).

## The modern additions

**`FrozenDictionary<TKey, TValue>` / `FrozenSet<T>`** — added in .NET 8. Read-only, *aggressively* optimised for lookup, slow to construct. Use these for lookup tables that are built once and read many times. The construction cost pays back the first few hundred lookups; for a static configuration map read on every request, the speedup is significant.

```csharp
private static readonly FrozenSet<string> AllowedRegions =
    new[] { "EU", "UK", "US" }.ToFrozenSet(StringComparer.OrdinalIgnoreCase);
```

**`ImmutableList<T>` / `ImmutableDictionary<TKey, TValue>`** — fully immutable; every mutation returns a new instance, sharing structure with the old. Use these when you need *value semantics* on a collection, when you are passing it across thread boundaries without locking, or when an audit-style *"the collection at time T"* semantic matters.

The cost is real — `ImmutableList<T>.Add` is O(log n), not O(1) — but the correctness properties are sometimes worth it. For the build-up phase before you freeze, reach for the matching `Builder`.

**`Span<T>` / `ReadOnlySpan<T>`** — not collections in the traditional sense, but the right tool for *"a window into a contiguous block of memory."* The single most underused tool in modern C# for performance-conscious code. Slicing a `Span<T>` allocates nothing. Iterating it is as fast as iterating an array.

```csharp
ReadOnlySpan<char> trimmed = input.AsSpan().Trim();
```

For string processing in hot paths, `Span<char>` and `ReadOnlySpan<char>` are transformative.

## The interface family

Three interfaces show up everywhere and are worth a moment.

**`IEnumerable<T>`** — *"you can foreach over this."* The most general contract. Says nothing about size, ordering, mutability, or whether iterating it is cheap (it may, behind the interface, be a database query that materialises lazily). The right return type for *"a sequence of things I may or may not enumerate."*

**`IReadOnlyCollection<T>`** — `IEnumerable<T>` plus a `Count`. The right return type when you want to promise *"this is a finite collection whose size I know."*

**`IReadOnlyList<T>`** — `IReadOnlyCollection<T>` plus indexing. The right return type when you want to promise *"this is an ordered collection I can index into."*

The mistake almost everyone makes — and I have made it for many years — is returning `IEnumerable<T>` from a method whose internal implementation is a `List<T>` whose count is known. The caller cannot ask for the count without enumerating. The caller cannot index without enumerating. The caller iterates twice and accidentally materialises a database query twice. *Return the most specific interface that is still honest.*

## Performance, in passing

The cost of choosing the wrong collection is rarely catastrophic in isolation. The cost is in the cumulative effect across a codebase. A few representative numbers, taken from informal benchmarks rather than published ones:

- `List<T>.Contains` on a 1,000-item list takes around five microseconds; `HashSet<T>.Contains` on the same data takes around fifty nanoseconds. The difference is roughly two orders of magnitude.
- `Dictionary<TKey, TValue>` lookup on a 10,000-entry dictionary takes around thirty nanoseconds; `FrozenDictionary<TKey, TValue>` lookup on the same data takes around ten. A threefold difference, sustained across every request.
- Building a `List<T>` of a million items with a capacity hint versus without is roughly a two-fold difference, almost all of it in GC pressure from the doubling allocations.

None of these is the difference between a working and a non-working application. All of them, applied consistently, are the difference between a snappy one and a sluggish one.

> *Reaching for `List<T>` regardless of access pattern is the algorithmic equivalent of using a bath towel for every job in the kitchen: it will get most things done, eventually, and the kitchen will smell faintly damp the entire time.*

## The smells

- `List<T>.Contains` in any code path that runs more than a few hundred times.
- `Dictionary<TKey, TValue>` whose key is a mutable class — equality may change after insertion, and the entry becomes unreachable.
- A method that returns `IEnumerable<T>` whose body builds a `List<T>` first.
- A `foreach` over a `List<T>` followed by `.ToList()` to materialise — the `foreach` already materialised it.
- An `ImmutableList<T>` built up by repeated `.Add()` calls — the builder pattern (`ImmutableList<T>.Builder`) is what you wanted.
- A `for (int i = 0; i < list.Count; i++) list.RemoveAt(i)` — you wanted `Clear()`, or a different collection entirely.

## Recap

- `T[]` for fixed-size known-in-advance. `List<T>` for dynamic build-up. `Dictionary<TKey, TValue>` for keyed lookup. The other types exist for specific reasons.
- `HashSet<T>` over `List<T>.Contains` is the most common single optimisation in real codebases.
- `FrozenDictionary<TKey, TValue>` and `Span<T>` are the modern additions worth knowing about.
- Return the most specific interface that is honest — `IReadOnlyList<T>` over `IEnumerable<T>` when the data really is a known list.
- The cost of choosing wrong is invisible per call and substantial cumulatively.

## Onwards

The next chapter is a short polemic, promised many chapters ago, about what other languages call *arrays* and what we as C# developers should appreciate, just for a moment, about the strongly-typed kind we get for free.
