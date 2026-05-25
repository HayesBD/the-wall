# Chapter 5 — Discriminated Unions: The Long-Awaited Guest

C# has had thirty years to get around to discriminated unions, and at the time of writing it is finally getting around to them. By the time you read this, they may have arrived in stable form; they may still be in preview; they may even, in some chronologies, be on their second or third iteration. The proposal has been worked on in public for several years and the language design team have been quite open about the journey. I will write this chapter as if they are landing, mark the preview-specific bits as such, and trust that anything time-locked will be obvious to a thoughtful reader.

The reason this feature has been so long awaited — and the reason it has taken so long — is that it sits at the intersection of two things C# already has and a third it never quite did. C# has classes (one shape, with members). C# has interfaces (a contract several shapes can implement). What C# has not had, until now, is a first-class way of saying *"a value is one of these specific shapes, here is the closed list, and the compiler will keep me honest about handling each one."*

That sentence is the entire pitch.

## What a discriminated union actually is

A discriminated union — DU for short — is a type whose value is exactly one of a small, named, closed set of variants, each of which may carry its own data. The compiler knows the full set, the compiler can prove you have handled every case, and the compiler will not let you forget when you add a new one.

A canonical example. Consider a payment method, which may be a credit card (with a number and expiry), or a bank transfer (with an IBAN), or simply cash (carrying no data of its own).

In a language that has had this for years — F# has, Rust has, Swift has, Scala has — you write something like:

```fsharp
type PaymentMethod =
    | CreditCard of number:string * expiry:DateOnly
    | BankTransfer of iban:string
    | Cash
```

Three variants. Each carries its own data, or none. The type system enforces that any function consuming a `PaymentMethod` must answer to all three.

C#'s preview syntax is in roughly the same key. I will not pin down the exact spelling, because by the time the ink is dry on this book it may well differ in small ways — the proposal has been through several drafts and the final form is still in motion. What matters is the shape: a closed list, named variants, optional per-variant data, and — crucially — exhaustiveness checking on the consumer side.

## Consuming a discriminated union

The pleasure of a DU is in the consumption. You pattern-match it:

```csharp
public decimal CalculateFee(PaymentMethod method) => method switch
{
    PaymentMethod.CreditCard cc   => cc.Number.StartsWith("4") ? 0.03m : 0.02m,
    PaymentMethod.BankTransfer bt => 0.005m,
    PaymentMethod.Cash            => 0m
};
```

Two things to note.

First, no `_ => throw` arm. The compiler can prove the cases are exhaustive, so the switch expression compiles without a default. If you add a new variant — `PaymentMethod.GiftCard`, say — and you do *not* update this switch, you get a compile error, not a runtime exception eight months later when an indignant customer attempts to pay with one.

Second, the per-variant data is bound by name in each arm. No casts, no nulls, no defensive checks. The compiler knows that if you matched the `CreditCard` arm, the `Number` and `Expiry` are present and correctly typed.

That combination — closed set, exhaustive matching, per-variant data — is what makes DUs such a powerful modelling tool. They let you say in code what the domain says in English: *"a payment method is either a credit card with these details, a bank transfer with this detail, or cash, and that is the entire list."*

## Until the preview lands — the fallback patterns

If you are reading this on a release of C# that has not yet shipped DUs, you have three serviceable workarounds. None is quite as nice as the real thing, but all of them give you most of the benefit.

**The sealed class hierarchy.** This is the closest in spirit to a DU, and the one you should reach for first.

```csharp
public abstract record PaymentMethod
{
    private PaymentMethod() { }

    public sealed record CreditCard(string Number, DateOnly Expiry) : PaymentMethod;
    public sealed record BankTransfer(string Iban) : PaymentMethod;
    public sealed record Cash : PaymentMethod;
}
```

A few small disciplines make this nearly as good as a DU: the abstract base has a private constructor (so no one can derive outside the file), the derived types are nested and `sealed` (so the list is closed), and they are records (so value equality comes for free). Pattern-matching consumers look almost identical to the DU version:

```csharp
public decimal CalculateFee(PaymentMethod method) => method switch
{
    PaymentMethod.CreditCard cc   => /* ... */,
    PaymentMethod.BankTransfer bt => 0.005m,
    PaymentMethod.Cash            => 0m,
    _ => throw new ArgumentOutOfRangeException(nameof(method))
};
```

The only real difference: you still need the `_ => throw` arm, because the compiler cannot prove exhaustiveness of an open inheritance hierarchy. Modern analysers can warn you, but they cannot, alas, forbid you.

**The OneOf library.** A small NuGet package that gives you `OneOf<T1, T2, T3>` types with switch helpers. Useful when you want DU-ish behaviour for ad-hoc combinations rather than named domain concepts. The cost: a dependency for what is, philosophically, a language feature.

**The discriminator-plus-cast.** An enum tag plus an `object` or `dynamic` field. Don't. The whole reason this chapter exists is so we never have to do this again.

## Where DUs shine

Once you have them in your toolkit, you start seeing the world in unions. A non-exhaustive list of places to reach for one:

- **Result types.** `Result<T, Error>` as either `Ok(T)` or `Err(Error)`. The end of throw-as-flow-control. (See [Chapter 36](../06_CodeThatDoesntSmell/36_TryCatch.md).)
- **Parse outcomes.** `ParseResult<T>` as either `Success(T)`, `SyntaxError(string)`, or `Incomplete`. The parser cannot return a thing-that-is-and-also-isn't.
- **State machines.** An `Order` that is variously `Draft(...)`, `Submitted(...)`, `Shipped(...)`, `Delivered(...)`, `Cancelled(...)`, each with the data appropriate to that state and *only* the data appropriate to that state. The end of nullable fields that are conditionally meaningful.
- **Command and event types.** A `BankingCommand` is either `Deposit(amount)`, `Withdraw(amount)`, or `Transfer(amount, target)`. The handler must answer for all three.

A useful diagnostic: any time you find yourself writing a class with a bunch of nullable fields where the comment says *"only set when status = X"*, you have a discriminated union pretending to be a class. Refactor.

## Performance, in passing

The DU preview's implementation is, by all accounts, designed to be zero-allocation for value-typed variants and cheap for reference-typed ones. The sealed-class fallback allocates one object per construction, which is the usual cost of references. Neither is meaningfully slower than the hand-rolled tag-and-cast alternative; both are dramatically safer.

> *Avoiding discriminated unions because the syntax is new is rather like refusing to fly until they invent a faster horse: a defensible position, technically, and one that does not age well.*

## The smells

- A class with five nullable fields, half of which are conditionally meaningful — you wanted a DU.
- An enum with a sibling `Dictionary<TheEnum, SomeData>` to attach per-value data — you wanted a DU.
- A method returning `(bool ok, T value, string error)` — you wanted `Result<T, Error>`, which is a DU.
- A switch expression with a `_ => throw` over a sealed hierarchy where you know all the cases — when the language gets full exhaustiveness checking, this becomes a compile-time guarantee. Until then, treat the `throw` as a placeholder.

## Recap

- A discriminated union is a closed list of named variants, each optionally carrying data, with exhaustive pattern matching on the consumer side.
- C#'s preview-feature DU brings this in proper form; sealed record hierarchies are the best workaround until it ships.
- DUs replace nullable-field-soup classes, enum-plus-data dictionaries, and tuple-of-flags return types.
- The end of the `_ => throw` is, slowly, in sight.

## Onwards

Next, the most underappreciated feature C# got right almost from the start: generics. The chapter also contains the polite raised eyebrow promised, several chapters ago, in the direction of languages that did not bother.
