# Chapter 18 — Metadata as a Component: The Quiet Superpower

There is a moment, somewhere around the third or fourth week of working in an ECS codebase, where a particular insight lands and quietly rearranges your thinking. I want to dedicate a short chapter to it, because for many readers — myself among them — it is the moment ECS stopped feeling like an arcane game-engine pattern and started feeling like the obvious answer to a problem they had been working around their whole career.

The insight is this: in an ECS world, *metadata is just another component*.

That sentence sounds, the first time it lands, faintly anti-climactic. It also takes a few examples to grow teeth. Let me give you the examples.

## The five-tonne base class problem

Picture a long-lived OO codebase. The kind we discussed at the end of Part II.

Somewhere near the root of the entity hierarchy, there is a class called something like `EntityBase` or `AuditableEntity` or `DomainEntity`. It contains the metadata fields every entity needs: `Id`, `CreatedAt`, `CreatedBy`, `LastModifiedAt`, `LastModifiedBy`, possibly a `RowVersion` for optimistic concurrency, possibly an `IsDeleted` for soft delete, possibly a `TenantId` for multi-tenancy, possibly a `SchemaVersion` for handling old shapes.

The base class started slim. Every entity in the system inherits from it. Every time a new cross-cutting concern arrives — tagging, suspension, archival, redaction — a debate begins about whether the new concern belongs on this base class (making it heavier and applying it to entities that do not want it) or on a new base class (which produces a tangle of multiple inheritance that C# does not, mercifully, allow) or on an interface (which gives you the *contract* without solving the *behaviour*) or as a separate table joined on the entity ID (which works but is an awful lot of ceremony for what was, originally, two timestamps).

Every option is, in some way, wrong. Every team in this situation has had the same meeting. I have personally sat in this meeting on three separate occasions, and the resolution was three different choices, each of which was regretted within the year.

## What the ECS world does instead

In an ECS world, there is no `EntityBase`. There is no debate. Each piece of metadata is its own component, attached to whichever entities care to have it.

```csharp
public record struct Audit(DateTimeOffset CreatedAt, UserId CreatedBy);
public record struct LastModified(DateTimeOffset At, UserId By);
public record struct SoftDeleted(DateTimeOffset At, UserId By, string? Reason);
public record struct Tenant(TenantId Id);
public record struct SchemaVersion(int Version);
public record struct Tags(IReadOnlyCollection<string> Values);
public record struct Suspended(DateTimeOffset At, string Reason);
public record struct OptimisticToken(Guid Value);
```

Each one is a small, focused, well-named piece of data. Each one applies to the entities that need it, and only those entities. Each one is added, removed, and queried with the same uniform API as every other component in the system.

The debate has not been won; the debate has been *dissolved*.

## What this unlocks

Five small consequences, each of which is worth a few hundred milliseconds of thought.

**Introspection becomes free.** *"Find every entity created by user X last Tuesday"* is no longer a custom report against the audit table. It is a query:

```csharp
foreach (var (entity, audit) in world.Query<Audit>())
{
    if (audit.CreatedBy == targetUser && audit.CreatedAt.Date == targetDate)
        yield return entity;
}
```

The same query shape works for every entity in the system that carries an `Audit` component, regardless of what *kind* of entity it is. The reporting code is generic. The audit *application* is generic. New entity kinds get audit support for free, the moment they receive the `Audit` component.

**Debugging becomes a query.** Want to know which entities were touched in the last hour? Add a `LastTouched` component to the systems whose behaviour you care about, run the application for an afternoon, query the world. You have not modified a single entity definition. You have not changed the database schema. You have not added a column anywhere. At the end of the day, remove the component and the system that added it, and nothing else changes.

I cannot overstate how often this kind of *temporary observability* is useful. In an OO codebase, the equivalent — adding a column, plumbing it through the ORM, updating the migrations, deploying, querying, then doing all of it in reverse — is a week of work that the team will, sensibly, decline to do for a *"just for now"* investigation.

**Schema migrations become additive.** When the `Customer` concept changes — gaining a `Tier`, gaining a `LoyaltyPoints` counter, gaining a `RegionalManager` reference — you add new components. The existing entities continue to work. The systems that care about the new components find them on the entities that have them; the systems that don't, don't. There is no *"migration of all customers to the new shape"*. The shape was never a single thing; the shape was always a *set of things*, and adding a new one is, by construction, additive.

**Cross-cutting concerns stop being cross-cutting.** The framing problem we identified at the end of Part II — *concerns that apply to a subset of entities chosen by some runtime rule* — has dissolved. Soft delete? A `SoftDeleted` component on whichever entities are soft-deleted, plus one filter system. Tenancy? A `Tenant` component, plus filter systems. Audit? An `Audit` component, set by an `AuditSystem` that listens for write events. Each concern is one component definition plus one system. Adding the concern to a new entity kind is *attaching the component*. Nothing else.

**Optionality becomes structural.** In the OO world, *"some customers have an associated regional manager"* becomes `RegionalManager? Manager` on the entity, with all the nullable-as-conditional-meaning nastiness we discussed under the heading of discriminated unions ([Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)). In the ECS world, customers with a regional manager carry a `RegionalManager` component; customers without one do not. The compiler does not need to model the absence. The systems that care iterate over the entities that have the component. The systems that do not care, do not iterate.

## A small worked example

The simplest illustration is the auditing concern, which I will write out at length not because it is sophisticated but because it shows how *small* the answer becomes once you stop fighting the base class.

```csharp
// The components.
public record struct Audit(DateTimeOffset CreatedAt, UserId CreatedBy);
public record struct LastModified(DateTimeOffset At, UserId By);

// The system.
public class AuditSystem
{
    public void OnEntityCreated(World world, EntityId entity, UserId actor)
        => world.Add(entity, new Audit(DateTimeOffset.UtcNow, actor));

    public void OnEntityModified(World world, EntityId entity, UserId actor)
        => world.Set(entity, new LastModified(DateTimeOffset.UtcNow, actor));
}
```

That is the entire feature. A handful of lines cover the audit concern for *every entity in the system, present and future*. New entity kinds get audited the moment something invokes `OnEntityCreated` for them. Old entity kinds can opt in retroactively by having the system run against the existing component set. No base class. No interface. No debate.

> *The first time a developer realises that the soft-delete feature they were dreading for sprint planning is, in ECS terms, a record struct and a six-line system, you can usually see them physically reconsider their last decade of choices.*

## What this is not (a small caveat)

I do not want to suggest metadata-as-component is a free lunch.

The discipline still applies. A codebase that adds a new metadata component for every passing whim will, just like an OO codebase that adds a new base-class concern for every passing whim, become hard to reason about — *which of these forty-seven components is the one that controls visibility in the API?* The cost is moved, not abolished. The cost being moved is, in practice, much easier to bear, because each component is small, named, and queryable in isolation; but it still wants discipline.

The catalogue of *what can go wrong* in this pattern is part of [Chapter 21](./21_ECSSmells.md), in due course.

## Performance, in passing

Metadata as a component is, almost without exception, faster than metadata as a base-class field. The reason is structural and was mentioned in passing in [Chapter 16](./16_ECSFirstLook.md): systems that don't care about a particular component never touch it, never materialise it, never serialise it, never include it in their iteration.

A `ReportingSystem` that wants a list of entities with their `Audit` data queries `Query<Audit>()` and gets exactly that: the audit components, contiguously, with no other component data along for the ride. The contrast with a base-class approach — which would have loaded the `Audit` fields on every entity the query touched, whether the reporting cared about them or not — is sometimes dramatic and is consistently in the right direction.

## Recap

- In ECS, metadata is not a special case. It is just another component.
- Audit, soft delete, tenancy, tagging, schema version, suspension — each becomes a small record struct plus, where needed, a small system.
- The base-class debates that haunt mature OO codebases simply do not arise.
- Introspection, debugging, and schema migration all become uniform operations.
- The pattern still wants discipline, but the cost of discipline is much smaller than the cost of the alternative.

## Onwards

The next chapter steps from *what components are* to *how they are stored*. The world, the registry, the type-driven dispatch that makes a `Query<Position, Velocity>` resolve to the right two arrays without the developer having to think about it. The plumbing is more interesting than it sounds — it is what makes the whole approach perform.
