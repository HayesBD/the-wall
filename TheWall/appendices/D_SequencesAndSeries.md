# Appendix D — Sequences, Series, and the Shapes Data Takes

[Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md) made the case that set theory underlies every query you write. It deliberately stopped at *sets*, because sets are where the querying lives. But sets have two close mathematical relatives — *sequences* and *series* — and the distinction between the three turns out to map, with pleasing exactness, onto three different things your code does with data every day. This appendix is a short tour of that mapping. It is not essential reading; it is for the reader who enjoyed the set-theory chapters and wondered what came next.

## Three mathematical objects

**A set** is a collection of distinct things in no particular order. `{3, 1, 4, 1, 5}` is, as a set, `{1, 3, 4, 5}` — the duplicate collapses, the order is irrelevant. We met this in Chapter 25.

**A sequence** is an *ordered* collection that *permits duplicates*. `(3, 1, 4, 1, 5)` is a sequence, and it is a different sequence from `(1, 3, 4, 5, 1)`, and the repeated `1` is a genuine, distinct member at its own position. A sequence has a *first* element, a *second*, an *nth*. It is, in essence, a function from the positions `1, 2, 3, …` to values.

**A series** is the *sum* (or, more generally, the accumulation) of the terms of a sequence. Given the sequence `(1, 2, 3, 4)`, the corresponding series is `1 + 2 + 3 + 4 = 10`. A series takes a sequence and folds it down to a single value.

Three objects: an unordered bag of distinct things, an ordered list that allows repeats, and the accumulation of such a list into one value. Hold those three shapes in mind, because your code is full of all three, usually without naming them.

## The mapping onto C#

Here is the correspondence, and it is exact enough to be slightly startling the first time you see it laid out.

| Mathematics | C# | Key property |
|---|---|---|
| Set | `HashSet<T>`, `FrozenSet<T>` | distinct, unordered |
| Sequence | `IEnumerable<T>`, `List<T>`, `T[]` | ordered, duplicates allowed |
| Series (the act of summing) | `Aggregate`, `Sum`, `Count`, `Average` | sequence folded to a value |

The single most consequential line in that table is the middle one, and it is the source of a confusion [Chapter 26](../04_DataAndSets/26_FiltersJoinsGroupings.md) flagged in passing: *`IEnumerable<T>` is a sequence, not a set.* It is ordered. It permits duplicates. When you call `.Distinct()`, you are converting a sequence into (the sequence form of) a set; when you call `.OrderBy(...)`, you are asserting an order that a true set would not have had; when you call `.Sum()`, you are computing a series. The LINQ surface is, in mathematical terms, a set of operations that move fluidly between these three objects, and knowing which object you are holding at each step explains a great deal of LINQ's behaviour.

## Why ordering is a property you opt into

A set has no order. A sequence does. This sounds academic until it bites, and it bites most often at the boundary between the two.

A `HashSet<T>` makes no promise about iteration order — and, crucially, the order it happens to iterate in *may change between runs, between framework versions, between machines*. Code that iterates a `HashSet<T>` and relies on the order is code with a latent, non-deterministic bug. If you need order, you are asking for a sequence, and you must say so — `.OrderBy(...)` on the way out, or a collection type that preserves order in the first place.

The reverse mistake — treating a sequence as though it were a set — shows up as the duplicate that was not supposed to be there. A `List<T>` happily holds the same element twice; if your logic assumed distinctness, the second copy is a bug waiting for the right input. The `.Distinct()` that converts the sequence to set-semantics is the fix, and forgetting it is one of the more common data bugs in business code.

The discipline: *be deliberate about which of the three shapes you are working with at each step*, because the shape determines what is and is not guaranteed.

## Lazy sequences and the infinite

Here is where sequences pull decisively ahead of sets in expressive power, and where C# does something genuinely elegant.

A mathematical sequence can be *infinite*: `(1, 2, 3, 4, …)` goes on forever. You obviously cannot store an infinite sequence in a `List<T>` — but you can *describe* one, and ask for as much of it as you need, because `IEnumerable<T>` is lazy. The `yield return` mechanism lets you define a sequence by its *rule* rather than its *contents*:

```csharp
public static IEnumerable<int> NaturalNumbers()
{
    var n = 1;
    while (true)            // infinite, and that is fine
        yield return n++;
}

var firstTen = NaturalNumbers().Take(10).ToList();   // 1..10
```

`NaturalNumbers()` is an infinite sequence. It does not loop forever when you call it, because nothing has asked it for a value yet — it is a *recipe*, in the deferred-execution sense of [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md). `Take(10)` asks for exactly ten values and then stops asking; the `while (true)` never runs an eleventh time. The Fibonacci numbers, the primes, the powers of two — any infinite mathematical sequence — can be rendered as a few lines of C# that compute only as much as the consumer pulls.

A *set* cannot do this. Membership in an infinite set is a predicate, not an enumeration; you cannot iterate the set of all primes. But you *can* iterate the infinite *sequence* of primes, lazily, taking what you need. The sequence's order is exactly what makes the laziness work — there is always a well-defined *next* value to compute.

## Series and the question of termination

The series — the fold of a sequence to a value — brings its own small piece of mathematical caution.

A finite series always terminates: `(1, 2, 3, 4).Sum()` is `10`, computed in four steps. But a series over an *infinite* sequence raises the question mathematicians call *convergence*. The series `1 + 1/2 + 1/4 + 1/8 + …` converges — it approaches `2` and never exceeds it, so you can compute it to any desired precision by taking enough terms. The series `1 + 2 + 3 + 4 + …` diverges — it grows without bound, and folding it is a computation that never finishes.

The code lesson is direct: `infiniteSequence.Sum()` is a bug — it will run until it overflows or the heat death of the universe, whichever your operations team notices first. `infiniteSequence.Take(n).Sum()` is the correct shape: bound the sequence to a finite window *before* folding it to a series. The `Aggregate` over a lazy sequence is only safe when something upstream has made the sequence finite.

This is the same caution as the divergent series, wearing a stack trace. A fold needs a finite input, or a termination condition, or it is not a computation — it is a hang.

## What this buys you in practice

Three concrete habits fall out of holding the distinction clearly.

**Choose the collection that matches the shape.** If the data is genuinely a set — distinct, order-irrelevant — use a `HashSet<T>` and get O(1) membership and automatic deduplication. If it is a sequence — ordered, possibly repeating — use a `List<T>` or `T[]`. Reaching for `List<T>` when you meant a set is the [Chapter 23](../04_DataAndSets/23_Collections.md) smell; reaching for `HashSet<T>` when order matters is its mirror.

**Use laziness deliberately.** A lazy `IEnumerable<T>` that computes on demand is the right tool for large or unbounded sequences, for pipelines where you may not consume everything, and for composing transformations without materialising the intermediate steps. It is the *wrong* tool when you will enumerate it more than once (each enumeration recomputes — see Chapter 27's deferred-execution trap), in which case materialise it with `.ToList()` first.

**Bound before you fold.** Any aggregation — `Sum`, `Count`, `Aggregate` — over a potentially infinite or very large lazy sequence must be bounded first. The fold is the series; the series needs a finite sequence; the `Take`, the `Where` that eventually exhausts, the pagination, is what makes it finite.

## A closing observation

The three shapes are not three different topics; they are three views of the same underlying thing — a collection of values — distinguished by what you are allowed to assume about it. A set lets you assume distinctness and forbids you order. A sequence grants you order and position and permits repeats. A series collapses the whole thing to a single accumulated value. Every collection type, every LINQ operator, every query you write is working with one of these three shapes at every moment, and the bugs cluster precisely at the points where the code assumed one shape and the data was another.

It is, like the set theory it extends, simpler than it sounds and more useful than it has any right to be.
