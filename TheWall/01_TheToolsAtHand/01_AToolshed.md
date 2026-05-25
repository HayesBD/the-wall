# Chapter 1 — A Brief Tour of the Toolshed

A confession to open with. For the first six years I wrote C# professionally, I wrote almost everything as a `class`. Domain entity? Class. View model? Class. Configuration object? Class. The little tuple of two integers I needed for ten lines of business logic? Class, naturally, with a constructor and two read-only properties and an `Equals` override I would inevitably get subtly wrong. If C# had been a workshop, I had been the carpenter who turns up each morning, surveys a wall of beautifully-arranged tools, and chooses, without fail, the same hammer.

This is not, strictly speaking, *wrong*. C#'s type system is sufficiently generous that you can do almost all of your work with classes and the runtime will not punish you for it. But the wall has rather more on it than the hammer, and several of the other tools turn out to be remarkably good at the jobs you have been using the hammer for. This chapter is a brief, deliberately unhurried walk along that wall.

## The five tools, and what each is actually for

C# gives us, in rough order of how often you should reach for them:

- **`class`** — a reference type, allocated on the heap, carrying identity. Two `class` instances with identical property values are not equal to each other unless you teach them to be. Mutable by default. Inherits from one base, implements many interfaces.
- **`record`** — a reference type with value semantics bolted on. Two records with identical property values *are* equal by default. Immutable by default. Built squarely for data.
- **`struct`** — a value type, allocated on the stack (mostly), with value semantics. Two structs with identical property values are equal. Cheap to allocate, expensive to copy if large. Cannot inherit from another struct.
- **`record struct`** — exactly what it sounds like: a struct with the records team's quality-of-life improvements. Value equality, deconstruction, `with`-expressions.
- **`enum`** — a tiny named integer. Cheap, primitive, deceptively dangerous (more on which in [Chapter 4](./04_Enums.md)).
- **`interface`** — not a type that *exists*; a contract that types *fulfil*. You do not allocate an interface. You allocate something that implements one.

I should add, before we go further, that the long-awaited **discriminated union** is on its way as a preview feature and will join the list properly once it lands. [Chapter 5](./05_DiscriminatedUnions.md) covers what it unlocks and what we use in the meantime.

## A worked example — the same little thing, five ways

Here is a humble piece of data: an amount of money, with a currency. The kind of thing every business application has. Watch what changes as we move it from tool to tool.

As a class:

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}
```

Cheerfully mutable. Two `Money` instances with the same amount and currency are not equal to one another. The `Currency` is a `string`, which means `"GBP"` and `"gbp"` and `"GPB"` and `""` are all permitted by the type system, and one of those is a future bug report. This is what a hammer-only workshop produces.

As a record:

```csharp
public record Money(decimal Amount, string Currency);
```

Same data. Immutable. Value equality. Deconstructable. `with`-expressable. Three lines did the work of fifteen. The `Currency` is still a string — we have not solved primitive obsession here, only thinned out the surrounding ceremony — but as a *shape*, this is a far smaller surface area and far harder to misuse.

As a record struct:

```csharp
public readonly record struct Money(decimal Amount, string Currency);
```

Same again, but allocated on the stack. For small, frequently-passed values, that is a real performance dividend — no GC pressure for the millionth invoice line of the day. The `readonly` keyword tells the compiler we mean it about immutability and lets it skip defensive copies. A larger struct would be a mistake here; `Money` has two fields and earns its place on the stack.

As a class behind an interface:

```csharp
public interface IMoney
{
    decimal Amount { get; }
    string Currency { get; }
}
```

Now we have separated *what `Money` does* from *how `Money` is implemented*, and you should ask, with some scepticism: was that worth doing? For a value object the answer is almost certainly no. For a *service* it is almost certainly yes. The distinction matters more than the syntax, and we will return to it in [Chapter 33](../05_Architecture/33_ServicesVsUtilities.md).

## Which one when

A working rule of thumb, offered with the usual disclaimer that rules of thumb are wrong roughly ten percent of the time:

- If the thing has **identity that persists** — a user, an order, a session — it is a `class`.
- If the thing is **data that describes** — a date range, a money, a coordinate, a result — it is a `record` or a `record struct`.
- If the thing is **small, copied often, and lives briefly**, prefer `struct` for the allocation savings. If you cannot honestly defend *small, copied often, lives briefly*, it is not a struct.
- If the thing is a **named member of a closed set** — status, type, role — reach for `enum` reluctantly and read [Chapter 4](./04_Enums.md) first.
- If you are describing a **contract that several types must honour**, that is an `interface`.

The mistake almost everyone makes — the one I made for six years — is reaching for `class` by reflex regardless of which of these descriptions actually fits. `class` is the most general tool. It will accept any of these jobs without complaint. It is also, for most of them, the wrong tool.

## Performance, in passing

A note that will recur throughout this series. The right type choice is, more often than not, also the cheaper one. Heap allocations are not free — every `new` is a small bill paid to the garbage collector, which the garbage collector will eventually return for. A class-shaped `Money` allocated millions of times in a tight loop is a measurable slowdown over a `record struct Money`. You will not notice it on the first request. You will notice it on the millionth, and you will spend an entire afternoon explaining to a perfectly nice colleague why the dashboard has gone red.

> *A class for everything is the architectural equivalent of buying a tractor to fetch the milk: it will absolutely do the job, and the journey will be a great deal more expensive than it needed to be.*

## The smells to watch for

A starter set, which the chapters on enums, the rich-vs-anemic debate, and model smells will expand in their own time:

- Every type in your domain is a `class` — you have a hammer-only workshop.
- A `class` whose only purpose is to hold five public properties — you wanted a `record`.
- A `struct` with twelve fields — you wanted a `class`; the JIT is doing copies you did not see.
- An `interface` with a single implementation, named identically to the implementation but prefixed with `I` — you have invented bureaucracy.
- An `enum` driving a `switch` whose cases each call quite different code paths — you wanted polymorphism, and possibly a discriminated union.

## Recap

- C# gives you classes, records, structs, record structs, enums, and interfaces. They are not interchangeable.
- Reach for the lightest tool the job tolerates.
- Value semantics belong on data; reference semantics belong on things with identity.
- The right choice tends to be the faster one. It is also, usually, the shorter one.

## Onwards

The next chapter takes the most quietly transformative tool on the wall — the record — and looks at what it changes about how we shape data. After that, the value-type tour continues with structs and the stack.
