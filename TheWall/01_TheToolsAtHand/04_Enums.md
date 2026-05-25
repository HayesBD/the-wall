# Chapter 4 — Enums, and Their Many Sins

The `enum` is, on the face of it, an unobjectionable little thing. A short list of named values for the small categorical decisions that pepper any program — order status, user role, file format, day of the week. There is no obvious harm in it. There is no obvious complexity in it. There is a strong argument that, having been one of the very first features in C# 1, it has earned a kind of veteran's pardon from any further scrutiny.

I would like, in this chapter, to revoke that pardon.

Not because the enum is wicked — it is not — but because it is the most over-used and least understood of the small types, and because a great many of the most common bugs I have seen across two decades of working on C# codebases trace, with little ambiguity, back to a misused enum.

## What an enum actually is

An enum is a *named integer*. That is the entire story.

```csharp
public enum OrderStatus
{
    Pending   = 0,
    Confirmed = 1,
    Shipped   = 2,
    Delivered = 3,
    Cancelled = 4
}
```

The compiler treats `OrderStatus.Confirmed` as the integer `1` with a label slapped on for human eyes. Under the hood, it is a four-byte integer. It allocates nothing. It compares as fast as an `int`. It serialises as cheaply as an `int`. It has no virtual methods, no fields, no instances, no behaviour at all beyond what its integer-ness gives it.

This is also, as we shall see, the source of all four of its sins.

## Sin the first — enums do not constrain their values

Despite appearances, the type system does not stop you from doing this:

```csharp
OrderStatus status = (OrderStatus)42;
```

This compiles. It runs. `status` now holds the integer `42`, of which there is no member by that name. Every `switch` you have written that handled `Pending`, `Confirmed`, `Shipped`, `Delivered`, and `Cancelled` will fall through to the default case, or to no case at all.

This is not paranoia. It happens, regularly, when deserialising user input, when round-tripping through a database that stored the integer rather than the name, when an older version of the application writes an enum value that a newer version has retired, when JSON arrives from a third party with a number you weren't expecting. The cure is defensive validation at every entry point, which everyone forgets to write, and `Enum.IsDefined` calls scattered throughout the codebase, which everyone forgets to write twice.

## Sin the second — enums have no behaviour

The moment an enum starts to drive *different code* for different values, the language begins to fight you.

```csharp
public decimal CalculateShipping(OrderStatus status, decimal subtotal) => status switch
{
    OrderStatus.Pending   => 0m,
    OrderStatus.Confirmed => subtotal * 0.05m,
    OrderStatus.Shipped   => subtotal * 0.07m,
    OrderStatus.Delivered => 0m,
    OrderStatus.Cancelled => 0m,
    _ => throw new ArgumentOutOfRangeException(nameof(status))
};
```

That is not unreasonable code on its own. The trouble starts when you have nine of these switch statements scattered across the codebase, each handling the enum slightly differently, and you add a new value — `OrderStatus.Returned`, say. The compiler will not tell you which switches need updating. The unit tests, if they did not happen to cover the new value, will not either. You will find the missing cases in production, one indignant customer email at a time.

In an object-oriented language, the answer to *"different behaviour for different values"* is polymorphism. Enums do not give you polymorphism. They give you a `switch`.

## Sin the third — enums drift from their consumers

The point above, generalised. Every `switch` over an enum is an implicit dependency on the enum's full membership. Add a value to the enum and you have changed the contract of every switch in the codebase, silently, without the compiler noticing. Subtract a value and the same is true in reverse.

Modern C# offers some relief: a `switch` expression returning a non-void value, with a `_ => throw` arm, will at least force the cases to evaluate at runtime. But it cannot force them to be *correct*. The smell remains.

## Sin the fourth — the `[Flags]` attribute is a small lie

You know the pattern:

```csharp
[Flags]
public enum Permissions
{
    None   = 0,
    Read   = 1,
    Write  = 2,
    Delete = 4,
    Admin  = 8,
    All    = Read | Write | Delete | Admin
}
```

The `[Flags]` attribute does not, contrary to what its name suggests, *make* the enum into a flag set. It tells `ToString()` and the debugger to render combined values nicely. The bitwise behaviour was always there; the attribute is cosmetic. What this gives you, in practice, is a type whose set of *valid* values is the powerset of the named ones, which is not a thing the compiler understands or can help you with. Bugs in `[Flags]` enums are some of the most frustrating to track down — they look right, the values look sensible, and the `&` you typed instead of `|` will not be caught by any tool short of a careful human review on a quiet day.

## When enums are the right answer

I have spent four sins criticising enums; in fairness, let me say where they shine.

- **Bounded, stable, behaviour-free categories.** The seven days of the week. The twelve months. The compass points. These do not grow. They do not have associated logic. An enum is exactly the right tool.
- **Interop with native APIs or wire formats** that expect integer constants. Enums are precisely the named-integer abstraction such APIs want.
- **Performance-critical inner loops** where the alternative would be a sealed class with virtual dispatch, and you have measured and confirmed that the dispatch cost is unacceptable.

For everything else — anything that grows, anything with behaviour, anything that crosses a trust boundary — there are better tools.

## The better tools

**The smart-enum pattern.** A sealed class with static readonly instances. Behaviour lives on the instances. Validity is enforced by construction.

```csharp
public sealed class OrderStatus
{
    public static readonly OrderStatus Pending   = new(nameof(Pending),   0m);
    public static readonly OrderStatus Confirmed = new(nameof(Confirmed), 0.05m);
    public static readonly OrderStatus Shipped   = new(nameof(Shipped),   0.07m);
    public static readonly OrderStatus Delivered = new(nameof(Delivered), 0m);
    public static readonly OrderStatus Cancelled = new(nameof(Cancelled), 0m);

    public string Name { get; }
    public decimal ShippingRate { get; }

    private OrderStatus(string name, decimal shippingRate)
    {
        Name = name;
        ShippingRate = shippingRate;
    }
}
```

`OrderStatus.Confirmed.ShippingRate` is now a switch-free property access. Adding a new status is one new line in this class. Every consumer of `OrderStatus` continues to compile and work.

**Discriminated unions** (see [Chapter 5](./05_DiscriminatedUnions.md)) are the right answer when each value carries different *data* in addition to different *behaviour*. A `PaymentMethod` that is either `CreditCard(string number, DateOnly expiry)` or `BankTransfer(string iban)` or `Cash` is a discriminated union, not an enum.

## Performance, in passing

Enums themselves are cheap — they are integers. The smart-enum pattern adds a small per-access overhead that is negligible outside of very tight loops. Discriminated unions, depending on their implementation, are free or near-free in the upcoming preview-feature form. In almost all real-world code, the performance cost of moving away from a primitive enum is unmeasurable. The cost of *keeping* a primitive enum that should have been a smart enum is measured, instead, in production incidents.

> *Reaching for an `enum` because it is the simplest tool to hand is rather like reaching for a butter knife to perform open-heart surgery: brisk, decisive, and unlikely to end well for the patient.*

## The smells

- A `switch` on an enum that has a `default` branch which throws — you are admitting the compiler cannot help you.
- The same `switch` written six times in different files over the same enum — you wanted polymorphism.
- A `[Flags]` enum with more than four members — the bug surface area has grown faster than you have.
- An enum value persisted as an integer in a database — you have coupled the database to the enum's *position*, not its *meaning*, and reordering the enum will silently corrupt data.
- An enum with a value called `Unknown` or `None` or `Other` — you have admitted the set was not, after all, closed.

## Recap

- Enums are named integers. They constrain nothing.
- A `switch` on an enum is a brittle contract with every other switch.
- Reach for the smart-enum pattern the moment behaviour appears.
- Reach for a discriminated union the moment per-value *data* appears.
- Persist enums as their names, not their integers, unless you enjoy data migrations.

## Onwards

The next chapter takes up the discriminated union proper — the long-awaited preview feature that finally promises C# a first-class answer to *"this value is one of several distinct shapes"*.
