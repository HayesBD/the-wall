# Chapter 3 — Structs and the Stack

The stack is, on the whole, faster than the heap. Allocation is the moving of a pointer rather than a request to the garbage collector. Deallocation is the moving of a pointer back. There is no bookkeeping. There is no compaction. There is no pause. For workloads that allocate furiously and deallocate immediately — the inside of a tight numerical loop, a high-throughput message pipeline, a per-frame game update — the difference between stack and heap is the difference between a polite request to the kitchen and a strongly worded letter to the local council.

The stack is also, in C#, where structs live.

Mostly.

## The mostly

A struct is a value type. When you declare a local struct variable, the compiler allocates it on the stack. When you copy it, the compiler copies the bytes. When the enclosing method returns, the stack frame goes with it and the struct is gone. The garbage collector is not involved at any point.

There are, however, asterisks. A struct that is a field on a class is allocated *inside* that class — on the heap, with the rest of the class's fields. A struct that is captured by a closure is allocated on the heap, inside the closure. A struct that is *boxed* — assigned to `object` or to an interface variable — is wrapped in a heap allocation, and that boxing is one of the more expensive things you can accidentally do.

So the more accurate phrasing is: a struct's storage location is whatever its container's storage location happens to be. Bare locals, parameters, and return values get the stack. Fields and captures inherit their container's home.

## When a struct is the right reach

A working rule, drawn from the wisdom of generations of grumpy .NET performance engineers:

**Use a struct when all three of the following are true.**

1. The data is **small**. The conventional guideline is sixteen bytes or thereabouts — roughly two `long`s, or a `decimal`, or four `int`s. Beyond that, copy costs begin to dominate the allocation savings.
2. The data is **logically a value**. A coordinate, a time range, a measurement, a small key. Not something with identity. Not something you would store in a database row.
3. The data is **immutable**. Always immutable. We will get to why in a moment.

When any of those three slips, use a class or a record.

## A worked example

A two-dimensional point, used inside a particle simulation:

```csharp
public readonly record struct Point(double X, double Y);
```

Two doubles. Logically a value. Immutable by virtue of being a `readonly record struct`. Used by the million in a per-frame update loop. This is exactly what structs were invented for.

A user account, holding a guid, three strings, a date, and a list of permissions:

```csharp
public class UserAccount { /* ... */ }
```

Not small. Has identity. Often mutates. Not a struct. Forcing it into one would not make it faster; it would make it considerably slower, because every method call passing a `UserAccount` would be copying every byte of it onto the stack again.

## The quiet horror of the mutable struct

C# permits you to write a mutable struct. It also permits you to put your hand in a wood-chipper, but the language not stopping you is not, on its own, a recommendation.

The problem is copying. Consider:

```csharp
public struct Counter
{
    public int Value;
    public void Increment() => Value++;
}

var counter = new Counter();
var counters = new List<Counter> { counter };
counters[0].Increment();
Console.WriteLine(counters[0].Value); // Prints 0
```

That last line is not a bug in C#. It is the inevitable consequence of struct copy semantics. `counters[0]` returns a *copy* of the struct stored in the list. `Increment` mutates the copy. The copy is discarded. The struct in the list is untouched. The reader of that code, debugging it on a Friday afternoon, will not enjoy themselves.

The fix is to make structs immutable. The `readonly struct` modifier enforces this at the compiler level and, as a bonus, allows the JIT to skip the defensive copies it would otherwise have to make to protect a possibly-mutating struct from itself:

```csharp
public readonly struct Counter
{
    public int Value { get; }
    public Counter(int v) { Value = v; }
    public Counter Increment() => new(Value + 1);
}
```

Immutable. `Increment` returns a *new* counter rather than mutating the old. This is the only safe shape for a struct.

## Boxing — the invisible heap allocation

The most expensive mistake you can make with a struct is to box it.

```csharp
Point p = new(1.0, 2.0);
object o = p;            // Boxing — heap allocation, copies the bytes into a wrapper.
IComparable c = someInt; // Also boxing.
```

Every assignment from a struct to `object` or to an interface allocates on the heap and copies the struct in. The performance you carefully bought by choosing a struct over a class is paid back, with interest, the moment you box.

Common sources of accidental boxing: storing structs in non-generic collections, calling methods that take `object` parameters, comparing a `Nullable<T>` against `null` in certain old forms, and the elderly `ArrayList` (which, if you find one in your codebase, deserves a moment of silence and a swift replacement).

The cure is, mostly, generics. The whole point of generics — which we will treat in [Chapter 6](./06_Generics.md) — is to give us type-safe collections and methods that *do not box* their value-type parameters. A `List<Point>` is fine. An `ArrayList` holding `Point`s is a small disaster.

## `ref struct` and the modern frontier

For completeness, since you will encounter them in the standard library: `ref struct` is a stricter form that promises *never* to be allocated on the heap, never to be captured by a lambda, never to be boxed, never to live longer than a stack frame. This is the machinery behind `Span<T>` and `ReadOnlySpan<T>`, which are themselves the machinery behind much of the modern .NET performance story.

We will not write our own `ref struct`s in this series — it is rarely the right level of abstraction for application code — but you should know they exist and that the standard library has stitched them into its hot paths.

## Performance, in passing

A small `readonly record struct` in a tight loop is one of the most lopsided performance trades available in C#: a few extra characters of syntax for the elimination of a great many GC allocations. Memory traffic falls. Pause times fall. The dashboard stays green. For data that meets the three criteria above, the choice is overwhelmingly obvious.

The reverse is also true. A large mutable struct passed by value through six methods is, in raw memory traffic, slower than the equivalent class — and harder to reason about into the bargain.

> *A four-hundred-byte mutable struct passed by value across a deep call stack is, performance-wise, the equivalent of getting on a flight in Singapore in order to post a letter from Heathrow.*

## The smells

- A mutable struct. Always. Without exception.
- A struct with twelve fields. Make it a class. Make it a record. Do not make it a struct.
- An interface variable holding a struct value — silent boxing, on every assignment.
- A `List<MyStruct>` whose elements are accessed with `list[i].SomeMutator()` — see the counter example above.
- A struct used to model something with identity. Identity belongs to classes.

## Recap

- Structs are value types, stack-friendly, copied on pass.
- Reach for them only when the data is small, logically a value, and immutable.
- `readonly` your structs so the compiler and the JIT can help you.
- Boxing undoes everything. Watch for it.

## Onwards

The next chapter takes the smallest, oldest, most underestimated type in the C# arsenal — the humble `enum` — and gives it the scrutiny it has, in my opinion, been spared for too long.
