# Chapter 16 — Entities, Components, Systems: A First Look

If you have spent your career writing business software in C#, the chances are very good that you have never written, or possibly even read, code organised around an Entity Component System. ECS — the abbreviation, like most abbreviations, is doing more work than it should — is a pattern that grew up in game engines, sometime around the middle of the noughties, where it solved a particularly thorny class of problem that game programmers had been wrestling with for years. Its widespread adoption in games was driven by Unity's *DOTS* in the late 2010s, by libraries like *EnTT* and *flecs* in the C++ world, and by *Bevy* in the modern Rust ecosystem.

It is not, on first encounter, an obvious fit for business software. The first time I described it to a colleague, he looked at me with the gentle pity reserved for a person who had clearly been working too many evenings.

A historical aside, because it bears on how you should read the word *new*. The idea that software should be built from *composable parts with declared relationships*, rather than from rigid hierarchies, is not new at all. In 1963 — the same era that produced the first object and relational ideas — Ivan Sutherland's *Sketchpad*, his MIT doctoral work and the ancestor of every CAD program since, let a user define a *master* object and stamp *instances* that referenced it, and declare *constraints* between elements that the system then maintained automatically as everything else changed. That is composition-over-duplication and declarative relationships, running on a computer the size of a room, sixty years ago — and mainstream CAD did not robustly catch up to Sketchpad's constraint-solving until the present century. ECS is not the same thing as Sketchpad; it is a different point on the same long arc, the one that bends away from rigid structure and towards composition. So when I call ECS a pattern you may not have met, I mean *newly relevant to business software*, not *newly invented*. The good ideas are frequently much older than their mainstream adoption — which is, if anything, a reason to take this one seriously rather than to wave it off as a fashion.

I want, in this chapter, to introduce the pattern as honestly as I can — what it is, where it comes from, what it gets you, and what it costs you. Part III as a whole will examine it with the same fair-minded scrutiny we applied to models in Part II. By the end of [Chapter 22](./22_ModelsVsECS.md), the reader will be in a position to make their own decision. This chapter is the entrance to the tour.

## The three primitives

ECS has exactly three concepts. They are small.

**An entity is an ID.** That is the entire definition. An entity is a unique identifier — usually an integer, sometimes a `Guid`, often wrapped in a `readonly record struct` for type safety — that exists for the sole purpose of being the thing that components are attached to. An entity has no fields, no methods, no behaviour, no shape. It is a handle. Nothing more.

```csharp
public readonly record struct EntityId(int Value);
```

**A component is data.** A component is a small, dumb, stateless piece of data that is *attached to an entity*. It has no methods of substance, no behaviour, no awareness of other components, no awareness of the world it lives in. It is, near enough, a `record`.

```csharp
public record struct Position(double X, double Y);
public record struct Velocity(double Dx, double Dy);
public record struct Name(string Value);
public record struct CustomerTier(TierLevel Level);
```

An entity can have any combination of components. A particular entity might carry a `Position` and a `Velocity`. Another entity might carry a `Position`, a `Name`, and a `CustomerTier`. A third might carry only a `Name`. The set of components an entity carries defines what the entity *is*, and that set can change at runtime.

**A system is behaviour.** A system is a stateless function (or class, in idiomatic C#) that operates on every entity carrying a particular set of components. A `MovementSystem` operates on every entity that has both a `Position` and a `Velocity`. A `LoyaltyPromotionSystem` operates on every entity that has both a `CustomerTier` and a `LifetimeSpend`. Systems do not know about each other. Systems do not hold state. Systems are pure transformations.

```csharp
public class MovementSystem
{
    public void Update(World world, double deltaTime)
    {
        foreach (var (entity, position, velocity) in world.Query<Position, Velocity>())
        {
            world.Set(entity, new Position(
                position.X + velocity.Dx * deltaTime,
                position.Y + velocity.Dy * deltaTime));
        }
    }
}
```

That, in three small definitions, is the entire conceptual model. Entities are IDs. Components are data. Systems are behaviour over component sets.

## The world

Tying these three together is a fourth thing, usually called the *world* or the *registry*, which is the container that owns the entities and the storage for components. The world is the moral equivalent of the `DbContext` in EF Core — the place where the universe of entities and their components actually lives.

The world's API is small and consistent:

```csharp
var entity = world.Create();
world.Add(entity, new Position(0, 0));
world.Add(entity, new Velocity(1, 1));

var hasPosition = world.Has<Position>(entity);
var position = world.Get<Position>(entity);
world.Remove<Velocity>(entity);
world.Destroy(entity);

foreach (var (e, pos, vel) in world.Query<Position, Velocity>())
    /* ... */;
```

That is, essentially, the public surface of an ECS framework. The implementation underneath can be quite sophisticated — we will come to component storage in [Chapter 19](./19_Registries.md) — but the API the application sees is small enough to fit on a postcard.

## A small worked example

To make the contrast clear, let me model the same small piece of a business domain two ways.

**The model approach.** A `Customer` class with a `Name`, a `Tier`, a `JoinedOn`, and (because the business person mentioned auditing last sprint) a `CreatedAt`, `CreatedBy`, `LastModifiedAt`, `LastModifiedBy`.

```csharp
public class Customer
{
    public CustomerId Id { get; private set; }
    public string Name { get; private set; }
    public TierLevel Tier { get; private set; }
    public DateOnly JoinedOn { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
    public UserId CreatedBy { get; private set; }
    public DateTimeOffset LastModifiedAt { get; private set; }
    public UserId LastModifiedBy { get; private set; }

    public void Promote(TierLevel newTier, UserId modifier) { /* ... */ }
}
```

So far so familiar. Now imagine, next sprint, the business asks for *soft delete* on customers. Then, the sprint after, *tagging* with arbitrary labels. Then, two sprints later, a *suspended* state with an associated reason and date. Each of these is a small request. Each lands as a new column on `Customer`, a new attribute, a new piece of logic in the constructor, a new method, an update to the four DTOs and the three view models, a small change to the audit logic.

After six months, `Customer` is six hundred lines and growing.

**The ECS approach.** A `Customer` is an entity with components.

```csharp
var alice = world.Create();
world.Add(alice, new Name("Alice"));
world.Add(alice, new CustomerTier(TierLevel.Standard));
world.Add(alice, new JoinedOn(new DateOnly(2025, 1, 1)));
world.Add(alice, new Audit(CreatedAt: now, CreatedBy: admin));
```

When the soft-delete request arrives, you add a `SoftDeleted` component, and a `SoftDeleteFilterSystem` that excludes entities carrying it from the normal queries. The `Customer` entity definition does not change. The five other entities in the system do not change. The existing systems do not change.

When the tagging request arrives, you add a `Tags` component and a `TagSearchSystem`. Again, no entity is "modified". The new behaviour attaches to whichever entities you choose to tag.

When the suspended-state request arrives, you add a `Suspended` component (carrying the reason and date), and a `SuspensionSystem`. Again, additive.

After six months, the *Customer entity definition* has not grown. What has grown is a collection of small, independent, well-named components and systems, each of which serves one concern and can be reasoned about in isolation. This is the property — *cheap, additive extension* — that is the headline claim for ECS and the reason this Part exists.

## Why this is not just composition

The natural reaction, on first seeing the example above, is to say: *that's just composition over inheritance. We already have that. I can put a `SoftDeleteInfo` property on `Customer` and a `Tags` collection and an `AuditInfo` and achieve the same thing.*

This is half right. ECS is, philosophically, composition over inheritance taken seriously. What makes it different from the *composition over inheritance* pattern of mainstream OO is two things.

**The components are not properties on a host class.** They are *first-class data*, stored separately from any notion of *the entity they belong to*. The entity is the ID; the components are stored in arrays or dictionaries keyed by entity ID. This sounds like a small implementation detail; it is, in fact, the structural reason ECS scales the way it does.

**The behaviour is not methods on a host class.** It is *systems that query for component combinations*. A `SoftDeleteFilterSystem` does not need to know what *kind* of entity it is filtering. Any entity in the world that carries a `SoftDeleted` component is in scope. Adding soft-delete to a new entity kind costs zero code in the new entity kind.

The combination of these two — components stored separately, systems querying component sets — is what gives ECS its characteristic *cost of extension*. Adding a feature touches the new components and the new systems and nothing else. The existing code does not change.

## What ECS does not solve

I have been generous in describing the upside. Let me, in the spirit of the series, be equally honest about the downside.

**Transactions.** Relational databases give you, for free, the ability to atomically modify several rows and either commit them all or commit none. ECS frameworks, by themselves, do not. Building a transactional layer on top of an ECS world is possible — we will discuss it in [Chapter 30](../04_DataAndSets/30_PersistingECS.md) — but it is work that the model approach got for free.

**Cross-entity queries with relational shape.** An ECS query is *"find every entity with these components"*. A relational query is *"find every order joined to its customer joined to its products"*. The first is straightforward; the second requires more thought in an ECS world, because the relationships between entities have to be modelled as components carrying entity IDs, not as foreign keys in tables.

**The familiarity tax.** Every mainstream business developer has been taught how to model with classes. Almost none have been taught ECS. Onboarding a new team member into an ECS codebase requires teaching the pattern, and the pattern is sufficiently different from what most developers know that this is a real cost, not a notional one.

**The premature complexity risk.** A CRUD admin tool with eight entities is the worst possible use case for ECS. The model approach handles it in an afternoon. The ECS approach handles it in a week, with no benefit to show for the extra time. We will return to this in [Chapter 22](./22_ModelsVsECS.md).

## Performance, in passing

ECS, in its game-engine heritage, comes with a particular performance story that is worth knowing about even if you never write a game.

Because components are stored separately from the entities they belong to — typically in tightly packed arrays per component type — iterating over every entity that carries a particular component is *extremely* cache-friendly. The CPU's prefetcher loves it. A `MovementSystem` updating ten thousand positions touches ten thousand `Position` records laid out contiguously in memory, with no pointer-chasing and no cache misses.

For business software, this raw-throughput story is rarely the headline. The performance benefit you actually feel in a business application is the more prosaic one: ECS systems naturally process only the entities that carry the components they care about, which means the work per request is bounded by the data that is actually relevant, not by the size of the entity. The god entity that materialised forty columns to answer a question about three of them is, structurally, the kind of waste ECS does not generate.

> *Comparing the cache behaviour of a tight ECS query to the page-fault opera of a deeply nested EF Core `Include` is rather like comparing a Boeing 787 to a hot-air balloon: both will, given enough time, get you to roughly the same place; only one of them will arrive on the day printed on the ticket.*

## Recap

- An entity is an ID. A component is data. A system is behaviour over component sets.
- The world is the container that owns entities and stores components.
- The headline property is *cheap, additive extension* — new components and systems do not modify existing ones.
- ECS does not, on its own, give you database transactions, relational joins, or developer familiarity. These are real costs.
- The performance story is real but not usually the primary reason to adopt the pattern in business software.

## Onwards

The next chapter dwells on the property that, in my view, is the central revelation of ECS: that systems are *stateless* and components are *flat data*, and that this combination — which feels initially like a betrayal of every object-oriented instinct — turns out to be liberating in ways the next several chapters will spend pages enumerating.
