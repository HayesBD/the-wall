# Chapter 17 — The Stateless Revelation

I want to focus, in this chapter, on the single property of ECS that took me longest to make peace with and that has, in retrospect, given me the most. It is the property that *components hold no behaviour* and *systems hold no state*.

The first time someone described this to me, my instinctive reaction — and I would bet that mine matches yours, reader — was to wonder what was left. If the data is just data and the behaviour is just behaviour and the two are not bundled into the same class, what exactly has been accomplished beyond a deeply old-fashioned procedural style with a fresh coat of paint?

The answer, which I propose to spend this chapter unpacking, is *quite a lot, actually.* The separation is not a step back from object orientation. It is a step *sideways* into a model that, once internalised, makes a series of difficult problems quietly easy. The chapter is mostly an enumeration of those problems and what happens to them when state and behaviour stop sharing a class.

## Why we have the instinct in the first place

Before we examine what we get, it is worth being explicit about what we are letting go of.

The instinct to put state on objects is not an arbitrary one. It comes from a real argument, made well, by the people who designed object orientation in the first place. The argument runs: data without the rules that govern it is data that can be put into invalid states; bundle the data with the rules and you guarantee the rules are always there to be enforced. *Encapsulation* is the name we give to that bundling, and for forty years it has been one of the two or three central ideas anyone learning to program has been taught.

I am not, in this chapter, going to suggest that idea was wrong. I am going to suggest it was *one* idea, and that there is a second idea — *separation* — that is equally valuable, and that the moment your codebase grows past a certain size the second idea begins to pay back more than the first.

## Components are flat data

In an ECS world, a component is, as we saw in [Chapter 16](./16_ECSFirstLook.md), a small record. No methods of substance. No behaviour. No awareness of the world it lives in or the other components attached to its entity. The strictness here is deliberate.

```csharp
public record struct Position(double X, double Y);
public record struct Velocity(double Dx, double Dy);
public record struct Health(int Current, int Max);
public record struct CustomerTier(TierLevel Level);
public record struct Audit(DateTimeOffset CreatedAt, UserId CreatedBy);
```

There is no `Position.MoveBy(...)` method on `Position`. There is no `Health.TakeDamage(...)` method on `Health`. There is no `CustomerTier.Promote()` method on `CustomerTier`. The data is there. The data is *only* there.

What this gives you, for free:

- **Trivial serialisation.** Every component is, by construction, something a JSON serialiser or a binary protocol will round-trip without help.
- **Trivial equality.** Two `Position`s with the same coordinates are equal by virtue of being records. No `Equals` override to maintain.
- **Trivial deep cloning.** A component has no references to walk, no event handlers to detach, no cyclic graph to flatten. A `with`-expression produces a perfect copy.
- **Trivial diffing.** Comparing the component state of an entity *now* to its state *a second ago* is a record-by-record equality check.
- **Trivial debugging.** Looking at an entity in a debugger shows you its components, in full, as plain data. There is no private state hidden behind a getter, no lazy-loaded thing waiting to fire, no internal cache that disagrees with the public surface.

If you have ever spent an afternoon trying to work out why a class instance's `ToString()` and its actual state disagreed, or why an `Equals` override compared the wrong fields, or why a JSON round-trip lost the value of `_internalCache`, you know what these properties are worth.

## Systems hold no state

The mirror image of *flat components* is *stateless systems*. A system is a class (or, in some ECS frameworks, simply a function) whose entire job is to query the world for entities with a particular component combination and do something with them. It owns no fields. It owns no cached state. It owns no per-entity bookkeeping. Whatever it needs, it reads from the world; whatever it produces, it writes back to the world.

```csharp
public class LoyaltyPromotionSystem
{
    public void Update(World world)
    {
        foreach (var (entity, tier, spend) in world.Query<CustomerTier, LifetimeSpend>())
        {
            var newLevel = DetermineTier(spend.TotalGbp);
            if (newLevel != tier.Level)
                world.Set(entity, new CustomerTier(newLevel));
        }
    }

    private static TierLevel DetermineTier(decimal spend) => spend switch
    {
        >= 10_000 => TierLevel.Platinum,
        >= 5_000  => TierLevel.Gold,
        >= 1_000  => TierLevel.Silver,
        _         => TierLevel.Standard,
    };
}
```

Notice what is not present: no field of any kind, no cache, no list of *which customers I have already promoted this run*, no logging hook smuggled in via constructor injection that decides midway through to skip an entity. The system is *what it does*, and what it does is fully visible in its public surface.

What this gives you:

- **Trivial unit testing.** A system is a pure transformation from `(world, inputs) → (world')`. Construct a world with three test entities, run the system, assert the resulting components. No mocks. No partial setups. No *"don't forget to also configure the FooMock or this test passes for the wrong reason"*.
- **Trivial parallelism.** Two systems that operate on disjoint sets of component types can run on different threads with no synchronisation. The ECS framework knows what each system reads and writes; the scheduler does the rest. Mainstream OO classes, with their hidden state and unconstrained collaborators, are vastly harder to parallelise this cleanly.
- **Trivial replay.** Snapshot the world's components at time T0. Run system A. Snapshot again at T1. If you ever need to know *exactly* what changed and why, you have the data. This is the foundation of deterministic debugging, time-travel inspection, and the kind of save/load functionality that game players take for granted.
- **Trivial hot reload.** Because systems hold no state, you can swap a system out for a new version mid-run without losing anything. The new version starts seeing entities on its next tick. There is no migration step.

## The objections, met fairly

There are two objections worth answering, because they are good ones and the rest of Part III depends on the answers landing.

**"But encapsulation!"** The argument: in OO, the private fields of a class are protected from the outside world; in ECS, every component is publicly accessible from every system. Has not encapsulation just been thrown away?

Yes and no. Encapsulation in the OO sense — *no code outside this class may touch these fields* — is, indeed, not the ECS model. But the underlying *goal* of encapsulation — *the rules about how this data may change live in one place* — is preserved. The rules just live in the *system* rather than on the *class*. The `CustomerTier` component can only be set by code that calls `world.Set<CustomerTier>(...)`, and a sufficiently disciplined codebase ensures that only `LoyaltyPromotionSystem` (or whichever other system explicitly owns that concern) does so. The encapsulation has moved up a level — from *per-class* to *per-concern* — but it has not vanished.

**"But invariants!"** The argument: in OO, the constructor of a class enforces invariants — *no `BankAccount` exists with a negative balance, because the constructor will throw*. In ECS, an entity has whatever components someone chose to attach, and there is nothing stopping a careless system from putting an account into a state the domain considers illegal.

This is the more serious objection. The answer is twofold. *First*, components themselves can still enforce invariants in their constructors — `new Money(-100, ...)` can throw exactly as it would in an OO codebase, because components are just records and records still have constructors. *Second*, cross-component invariants (*"an account with a `Closed` component cannot also have a `PendingTransfers` component"*) are enforced by the *systems* whose job is to enforce them — a `ClosureSystem` that, when it adds a `Closed` component, also removes `PendingTransfers`. The discipline is real, and we will treat it properly when we get to ECS smells in [Chapter 21](./21_ECSSmells.md). What is gained, in exchange for that discipline, is that the invariant logic lives in one well-named system rather than smeared across the entity class and the services that touch it.

## A worked transformation

Take the `Customer.Promote(newTier, modifier)` method from the previous chapter and watch what it becomes.

**The model version:**

```csharp
public void Promote(TierLevel newTier, UserId modifier)
{
    if (newTier == _tier)
        return;
    if (newTier < _tier)
        throw new InvalidOperationException("Tier cannot be downgraded via Promote.");
    _previousTier = _tier;
    _tier = newTier;
    _lastModifiedAt = DateTimeOffset.UtcNow;
    _lastModifiedBy = modifier;
    DomainEvents.Raise(new CustomerPromoted(Id, _previousTier, newTier));
}
```

Behaviour on the entity. Mutates several private fields. Raises a domain event from inside the entity. Touches static state to do so.

**The ECS version:**

```csharp
public class CustomerPromotionSystem
{
    private readonly IEventBus _events;

    public CustomerPromotionSystem(IEventBus events) => _events = events;

    public void Promote(World world, EntityId customer, TierLevel newTier, UserId modifier)
    {
        var currentTier = world.Get<CustomerTier>(customer);
        if (newTier == currentTier.Level)
            return;
        if (newTier < currentTier.Level)
            throw new InvalidOperationException("Tier cannot be downgraded via Promote.");

        world.Set(customer, new CustomerTier(newTier));
        world.Set(customer, new LastModified(DateTimeOffset.UtcNow, modifier));

        _events.Publish(new CustomerPromoted(customer, currentTier.Level, newTier));
    }
}
```

The behaviour has moved off the entity onto the system. The system's dependencies — the event bus — are explicit constructor parameters. The state lives entirely in the world. The test fixture for this system is *one world plus one event-bus fake*, with no fixture for `Customer` itself, because `Customer` no longer exists as a class.

I will not pretend this is shorter. It is not. What it is, is *flatter*, *more explicit*, and — for the cost of a small amount of additional ceremony per operation — utterly free of the hidden coupling that turns the model version into a maintenance hazard at scale.

## Performance, in passing

The two properties — flat components, stateless systems — combine into a performance story that is not, strictly speaking, the *reason* to adopt ECS in business software, but is a happy by-product.

Flat components store well. A `Position[]` of ten thousand entries laid out contiguously in memory is cache-friendly in a way no array of class references can ever be. Stateless systems iterate well. A system that touches only `Position` and `Velocity` can be vectorised, parallelised across cores, and JIT-optimised to a tighter inner loop than the equivalent OO method-with-mutation can be.

For business software, the wins are usually smaller and quieter — fewer allocations per request, lower change-tracker overhead, less hidden work — but they are consistently in the same direction. Cleaning up the structural coupling of a model codebase tends to make it faster as a side effect; rebuilding the same logic as ECS systems tends to make it *significantly* faster, without anyone setting out to.

> *Asking* "is this fast?" *of a tight ECS system feels rather like asking* "is the bicycle aerodynamic?" *of a piece of standard-issue racing gear: the question is not wrong, but the answer is so consistently* yes *that the asking has become a small ritual rather than a useful diagnostic.*

## Recap

- Components in ECS are flat data with no behaviour. This gives you trivial serialisation, equality, cloning, diffing, and debugging.
- Systems are stateless transformations from world to world. This gives you trivial testing, parallelism, replay, and hot reload.
- The OO instinct to bundle state with behaviour is preserved at a coarser grain: encapsulation moves from the class to the concern.
- Invariants survive — component constructors still throw; cross-component invariants live in the systems that own them.
- The performance story follows from the structural one and is, in business software, usually quiet but consistent.

## Onwards

The next chapter is the moment of revelation for many readers — the realisation that, once your components are flat data and your systems are queries, *metadata* is no longer a special-case bolt-on but simply *another component*. Created-at, created-by, soft-delete, tags, dirty markers, schema versions — all of them stop being awkward base-class concerns and start being first-class queryable data. The chapter is short. The implications are large.
