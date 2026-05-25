# Chapter 2 — Records: The Quiet Revolution

You may not remember, but there was a time before records. It was not a particularly long time ago — they arrived in C# 9, around the back end of 2020 — but the years before them have a particular flavour in the memory: the era of the data-carrier class with the constructor, and the read-only properties, and the override of `Equals`, and the override of `GetHashCode`, and the override of `ToString`, and the explicit copy-with-changes method you had to write yourself, by hand, every time. A typical immutable data carrier in C# 8 was a great deal of mechanical ritual surrounding a small number of actual fields.

Records changed that, and they did so without much fanfare. There was no keynote demoing records to a roomful of clapping people. They simply arrived in the language and within a year had begun to quietly reshape how we model data.

More than once, looking at a particularly hopeless old DTO, I have caught myself rewriting it as a record with the kind of muted satisfaction usually reserved for tidying a sock drawer.

## What a record actually is

A record is a *reference type with value semantics bolted on by the compiler*. That single sentence carries most of what matters, but let us unpack it slowly.

A reference type means: allocated on the heap, accessed via reference, eligible for null. So far, identical to a class.

Value semantics bolted on means: two records with equal property values are considered equal to each other, even though they are different objects on the heap. This is what classes do not do without considerable effort on your part.

Bolted on by the compiler means: you write nothing. The compiler emits the equality, the hash code, the `ToString`, the deconstruction, the copy constructor, and the `with`-expression support. It is not magic; it is generated code. If you ever want to see it, the C# compiler will happily show you.

## The two flavours

There are two ways to declare a record:

```csharp
public record PositionalMoney(decimal Amount, string Currency);
```

That is a **positional record**. The parameters are also properties, also `init`-only, also part of the equality contract — three things at once. This is the shape you will use most of the time.

```csharp
public record PropertyMoney
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }
}
```

That is a **property-style record**, written more like a class. It works identically. Reach for this form when you need attributes on individual properties, when the parameter list would be unwieldy, or when object-initialiser syntax reads better at the call site.

The two compile to broadly the same thing. Choose by readability.

## The `with`-expression

Records bring with them a small piece of syntax that, on first sight, looks like a curiosity and on the second sight looks like the thing you have been wanting your whole career:

```csharp
var gbp = new Money(99.99m, "GBP");
var converted = gbp with { Currency = "USD" };
```

`converted` is a fresh `Money`, identical to `gbp` except for the `Currency`. The original is untouched. The new value is built in one expression with no mutation in sight. For a domain object whose evolution you want to track through a series of immutable snapshots — a stage in a workflow, a step in a wizard, an entry in an audit trail — this is transformative.

## Value equality, in some detail

Records consider themselves equal to one another if and only if all their declared properties are equal. This is the bit that makes them genuinely useful, and also the bit that catches the unwary.

```csharp
var a = new Money(10m, "GBP");
var b = new Money(10m, "GBP");

Console.WriteLine(a == b);                             // True
Console.WriteLine(a.Equals(b));                        // True
Console.WriteLine(a.GetHashCode() == b.GetHashCode()); // True
Console.WriteLine(ReferenceEquals(a, b));              // False
```

That last line is the trap. `a` and `b` are *equal* but not *the same object*. For data, that is exactly what you want. For an entity with identity — a `User` with a database row — it is exactly what you do not want. A `User` with `Id: 5` and another `User` with `Id: 5` should be the same regardless of whether their `LastLoginAt` properties happen to match, and a record will cheerfully tell you they are different the moment any property diverges. This is one of the reasons we will spend a good part of Part II arguing that records are excellent for *data* and poor for *identity*.

## Inheritance, briefly

Records can inherit from other records. They do this rather more carefully than classes — the equality contract extends across the hierarchy, the compiler-generated `EqualityContract` property keeps subtypes distinct, and the whole thing works in a way that is genuinely impressive when you look at it.

You should still, in almost all cases, prefer composition. Records were not designed to make inheritance pleasant. They were designed to make data pleasant. Reaching for record inheritance is usually a sign that you wanted a discriminated union (see [Chapter 5](./05_DiscriminatedUnions.md)) and have not realised it yet.

## Performance, in passing

A record is still a reference type. Every `new Money(...)` is still a heap allocation. The value-equality machinery is fast — the compiler emits straight-line comparison code — but it is not free, particularly for records with a large number of properties.

If allocation pressure matters and the data is small, reach for `record struct`. The syntax is identical with one extra keyword:

```csharp
public readonly record struct Money(decimal Amount, string Currency);
```

You now have value semantics *and* stack allocation. The next chapter, on structs proper, has more to say about when this is the right reach and when it is, gently, not.

## The smells

- A `record` representing a database entity with an `Id` — you have wanted value equality and have got it on the wrong axis. Identity, not properties.
- A `record` with a public `set;` rather than `init;` — you have written a mutable record, which is technically permitted and conceptually a betrayal of everything records are for.
- A deep `record` inheritance hierarchy — you wanted a discriminated union.
- A flock of records each wrapping a single `decimal` and a `string` for a currency — you wanted a value object. We will treat this properly under primitive obsession.

> *Writing a forty-line DTO class in 2026 is the programming equivalent of choosing, on a sunny Tuesday, to walk to work via the gravel pit.*

## Recap

- A record is a reference type with value equality, immutability, deconstruction, and `with`-expressions generated for you.
- Reach for positional records by default; reach for property-style when attributes or initialiser syntax help.
- Records are for *data*, not for *identity*.
- For small, frequently-passed values, prefer `record struct` for the allocation savings.

## Onwards

The next chapter turns to the stack and to structs in their non-record form: when value-type semantics are the right reach, when they are the wrong one, and the quietly frightening business of large struct copies.
