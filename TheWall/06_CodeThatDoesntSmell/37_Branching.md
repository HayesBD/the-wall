# Chapter 37 — Branching: The Silent Killer

Every `if` statement is a small tax on the reader.

I want to begin with that claim stated baldly, because it sounds like an exaggeration and it is not one. Each branch a method contains doubles, in principle, the number of paths through it that a reader must hold in mind to be sure they understand it. A method with no branches has one path. A method with three independent `if`s has eight. A method with ten has over a thousand. The reader cannot, of course, hold a thousand paths in their head, which is precisely the problem: past a fairly low threshold, a heavily-branched method stops being something a human can fully reason about and becomes something they can only hope about.

This chapter is about recognising branches as a cost, and about the surprising number of them that turn out to be avoidable.

## The type switch — branching's most common smell

The single most common avoidable branch is the *switch on a type or a type-like value*, performing different behaviour per case.

```csharp
public decimal CalculateShipping(Parcel parcel)
{
    if (parcel.Type == ParcelType.Standard)
        return parcel.Weight * 0.5m;
    else if (parcel.Type == ParcelType.Express)
        return parcel.Weight * 1.5m + 5m;
    else if (parcel.Type == ParcelType.Fragile)
        return parcel.Weight * 0.5m + 10m;
    else if (parcel.Type == ParcelType.International)
        return parcel.Weight * 2m + 15m;
    else
        throw new ArgumentOutOfRangeException();
}
```

This is the smell we first met under enums in [Chapter 4](../01_TheToolsAtHand/04_Enums.md). The trouble is not this one method; it is that there are usually *several* such switches scattered across the codebase — one for shipping, one for labelling, one for insurance, one for tracking — each switching on the same `ParcelType`, each needing to be updated when a new parcel type is added, and none of them telling you about the others. Add `ParcelType.Refrigerated` and you have four (or fourteen) switches to find and update, with the compiler offering no help in locating them.

There are three good ways out, in rough order of how often each is the right reach.

## Way out one — polymorphism

If the thing being switched on is genuinely a *kind of thing* with *kind-specific behaviour*, the object-oriented answer is to let each kind carry its own behaviour.

```csharp
public abstract class Parcel
{
    public decimal Weight { get; init; }
    public abstract decimal CalculateShipping();
}

public class StandardParcel : Parcel
{
    public override decimal CalculateShipping() => Weight * 0.5m;
}

public class ExpressParcel : Parcel
{
    public override decimal CalculateShipping() => Weight * 1.5m + 5m;
}
```

The switch is gone. `parcel.CalculateShipping()` dispatches to the right implementation automatically. Adding `RefrigeratedParcel` is a new class with the one method, and — crucially — the compiler will require that class to implement `CalculateShipping`, so the new type *cannot* be added without supplying its shipping behaviour. The thing the enum switch could not enforce, the type system now enforces for free.

The caveat, and it is the same one from the rich-vs-anaemic debate of [Chapter 11](../02_TheModelQuestion/11_RichVsAnemic.md): this works cleanly when the behaviour belongs *on* the type. When the behaviour belongs elsewhere — when `CalculateShipping` needs a pricing service the parcel should not know about — polymorphism on the parcel is the wrong reach, and one of the next two ways out is better.

## Way out two — pattern matching with exhaustiveness

When the behaviour does *not* belong on the type, the modern C# answer is the switch expression — which is still a branch, but a far better-behaved one than the `if`/`else` chain.

```csharp
public decimal CalculateShipping(Parcel parcel) => parcel switch
{
    StandardParcel p      => p.Weight * 0.5m,
    ExpressParcel p       => p.Weight * 1.5m + 5m,
    FragileParcel p       => p.Weight * 0.5m + 10m,
    InternationalParcel p => p.Weight * 2m + 15m,
    _ => throw new ArgumentOutOfRangeException(nameof(parcel))
};
```

This is denser, the cases are aligned and scannable, and — when the type being matched is a sealed hierarchy or a discriminated union (see [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)) — the compiler can check the match for *exhaustiveness*, warning or erroring if a case is missing. The `_ => throw` becomes a backstop the compiler can tell you is unreachable rather than a runtime gamble. This is the branch tamed: still a branch, but one the type system helps you keep correct.

## Way out three — the lookup table

For the common case where the branch is really *a mapping from a key to a value or a function*, the cleanest answer is often not a branch at all but a dictionary.

```csharp
private static readonly Dictionary<ParcelType, Func<Parcel, decimal>> ShippingRules = new()
{
    [ParcelType.Standard]     = p => p.Weight * 0.5m,
    [ParcelType.Express]      = p => p.Weight * 1.5m + 5m,
    [ParcelType.Fragile]      = p => p.Weight * 0.5m + 10m,
    [ParcelType.International] = p => p.Weight * 2m + 15m,
};

public decimal CalculateShipping(Parcel parcel) => ShippingRules[parcel.Type](parcel);
```

The branching has been replaced by *data*. The rules are a table you can read at a glance, extend with one line, and — if the requirement arises — load from configuration rather than code. The control flow is now a single lookup with no conditional at all. For mappings that are genuinely tabular — *this key produces this behaviour* — this is frequently the clearest of the three, precisely because it stops pretending to be logic and admits that it is data.

## Boolean blindness and the flag parameter

A particular branching smell worth naming on its own: the *boolean parameter that selects behaviour*, which we first met in [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).

```csharp
public void Export(Report report, bool asPdf, bool includeHeaders, bool compress)
```

A call site reads `Export(report, true, false, true)` and tells the reader nothing — three booleans, no names, a small puzzle to decode. Worse, inside the method, each boolean is a branch, and three booleans are eight combinations, of which perhaps three are ever actually used and the other five are untested nonsense that nonetheless compiles. *Boolean blindness* is the condition of having reduced a meaningful choice to a bare `true`/`false` that has lost its meaning at the call site.

The cures: an enum where the choice is *which of several*, separate methods where the choice is *whether to*, an options record where there are genuinely several independent toggles. Each replaces the unreadable positional boolean with something that says what it means.

## When an `if` is the right answer

I have spent the chapter arguing against branches, so let me be clear about when one is correct — because the goal is fewer *unnecessary* branches, not zero branches through some misplaced purity.

A single `if` expressing a genuine, local, binary decision is perfectly good code. *"If the cart is empty, show the empty state; otherwise show the items"* is an `if`, and dressing it up as a polymorphic hierarchy or a lookup table would be the over-engineering the series has warned against throughout. The guard clauses of [Chapter 35](./35_EarlyReturns.md) are `if`s, and they are the right tool.

The branches worth eliminating are the *repeated* ones (the same switch in six places), the *type-dispatching* ones (behaviour selected by a kind-of-thing check), and the *combinatorial* ones (several booleans multiplying into untested paths). A lone, well-named, local `if` is not a smell. It is just an `if`, and most methods will have a few.

## Performance, in passing

The performance story here is mild and mostly favourable, with one caveat worth knowing.

Polymorphic dispatch is a virtual call — one indirection through a vtable — which the JIT can sometimes devirtualise and inline when it can prove the concrete type, and sometimes cannot. A `switch` expression on a type compiles to a series of type checks, which for a handful of cases is comparable. A dictionary lookup is a hash and a fetch. For the overwhelming majority of code, all three are far below the threshold of mattering, and you should choose on clarity.

The one genuine performance caveat runs the other way: in an extremely hot inner loop — the kind of place [Chapter 3](../01_TheToolsAtHand/03_Structs.md) and the ECS performance discussions cared about — a tight `switch` on an enum can actually be *faster* than virtual dispatch, because it avoids the indirection and the JIT can lay it out as a jump table. So in the rare hot-loop case, the branch you were trying to eliminate may be the right thing to keep. As ever: clarity by default, measure before optimising, and do not let either concern bully the other in the cases where it does not apply.

> *A method with a dozen independent boolean branches does not have a dozen behaviours; it has over four thousand, of which the author has tested perhaps five and the remaining several thousand are sitting quietly in production like unexploded ordnance from a war nobody remembers declaring.*

## The smells

- The same `switch`/`if`-chain on the same value appearing in more than one place — you wanted polymorphism, a discriminated union, or a lookup table.
- A `switch` with a `default: throw` over a closed set of types — the exhaustiveness the compiler could check if the set were sealed.
- A method taking two or more boolean parameters — boolean blindness; the call site is a puzzle.
- A nested ternary more than one level deep — write it as a `switch` expression or pull it apart.
- An `if`/`else` chain mapping keys to values — it wanted to be a dictionary.
- A branch count (the number of independent decisions in a method) high enough that you cannot enumerate the paths — the method wants splitting.

## Recap

- Every branch is a tax on the reader; the paths through a method multiply with each independent decision.
- Type-dispatching switches are the most common avoidable branch — replace with polymorphism, pattern matching, or a lookup table.
- Boolean parameters cause boolean blindness; prefer enums, separate methods, or options records.
- A lone, local, well-named `if` is not a smell — eliminate the *repeated*, *type-dispatching*, and *combinatorial* branches, not all of them.
- Clarity by default; the rare hot loop may justify keeping the branch.

## Onwards

The final chapter of this Part gathers everything — the model smells of Part II, the ECS smells of Part III, the branching and exception and nesting smells of this Part, and a dozen more the series has not yet had room to name — into one consolidated bestiary, organised for reference, with each smell's symptom, cause, and cure in one place.
