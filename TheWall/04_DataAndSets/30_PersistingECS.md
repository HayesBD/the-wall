# Chapter 30 — Persisting an ECS World: Where Components Meet Relational Tables

Traditional ORMs were built around a comfortable assumption: that an *entity* is a *class* is a *table*, and that an instance is a row. EF Core, Hibernate, Active Record, and the rest all rest on this triple equivalence, and it serves them well because, for the model-shaped applications they were designed for, the assumption is true.

ECS does not believe a word of it. In an ECS world, an entity is an ID, a component is data attached to that ID, and the *shape* of an entity is the *set of components it currently carries* — a set that changes at runtime. There is no class. There is no fixed shape. The triple equivalence the ORM depends on has been quietly dissolved.

So the question that has been hanging since [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md) is a real one: *how do you save an ECS world to a relational database without surrendering either the relational guarantees you want or the component flexibility you came to ECS for?*

There are three honest answers. Each has trade-offs. We will take them in turn.

## Pattern one — table per component type

The most relationally pure approach. Each component type gets its own table, keyed by entity ID.

```
Entities:        (Id)
NameComponents:  (EntityId, Value)
TierComponents:  (EntityId, Level)
AuditComponents: (EntityId, CreatedAt, CreatedBy)
TagComponents:   (EntityId, Tag)        -- multiple rows per entity
```

An entity that carries `Name`, `Tier`, and `Audit` has one row in `Entities`, one in `NameComponents`, one in `TierComponents`, and one in `AuditComponents`, all sharing the same `EntityId`. An entity that carries only `Name` has a row in `Entities` and a row in `NameComponents` and nothing else.

This maps beautifully onto EF Core. Each component table is an entity type with an `IEntityTypeConfiguration<T>` (the pattern from [Chapter 29](./29_DbContextConfiguration.md)). The `EntityId` is the foreign key back to the `Entities` table. Reconstructing an entity is a set of left joins — *"give me everything in every component table for entity 42"* — and the absence of a row in a given component table simply means the entity does not carry that component.

**The strengths.** SQL stays completely familiar. Ad-hoc reporting is straightforward — *"every entity with a `Tier` of Platinum"* is a single-table query against `TierComponents`. The relational integrity tools — foreign keys, constraints, indexes — all work exactly as they always did. A DBA can understand the schema without learning ECS.

**The cost.** Reconstructing a full entity with twelve components is a twelve-way join (or twelve separate queries). For systems that frequently need the *whole* entity, this is more expensive than the single-row read a model-based schema would have offered. The mitigation is the same as everywhere in Part IV: *project only the components you need*, exactly as [Chapter 27](./27_LinqAndEfCore.md) argued for projecting only the columns you need. A system that cares about `Tier` queries only `TierComponents`. The twelve-way join is only paid when twelve components are genuinely wanted.

## Pattern two — core table plus component bags

Fewer joins, achieved by storing components as a serialised collection — either in a JSON column or as EF Core owned types.

```
Entities: (Id, ComponentsJson)
```

where `ComponentsJson` is a JSON document like:

```json
{
  "Name": { "Value": "Alice" },
  "Tier": { "Level": "Platinum" },
  "Audit": { "CreatedAt": "2025-01-01T00:00:00Z", "CreatedBy": "admin" }
}
```

Modern SQL Server and PostgreSQL both support JSON columns with *queryable* contents — you can index into the JSON, filter on its values, and (with care) build indexes over JSON paths. EF Core's JSON-column support, mature since EF Core 7-8, maps this onto strongly-typed component objects on the C# side.

**The strengths.** Reconstructing an entity is a single-row read — the whole component bag arrives together. No joins. Adding a new component type requires no schema migration; the new component simply appears in the JSON of entities that carry it. This is the closest a relational store comes to ECS's *additive extension* property.

**The cost.** The JSON column is opaque to most relational tooling. The lovely *"every entity with a Platinum tier"* query that was a one-liner in Pattern One becomes a JSON-path query — supported, but less efficient and less universally understood. Reporting tools, BI connectors, and the average DBA are less comfortable with JSON-in-a-column than with honest tables. And while modern engines index JSON paths, the indexing story is less mature and less automatic than for ordinary columns.

## Pattern three — event-sourced component changes

The most sophisticated, and the right fit for a specific kind of system. Instead of storing the *current* state, store the *log of changes* that produced it.

```
Events: (Id, EntityId, ComponentType, Operation, Payload, OccurredAt)
```

Each row is a single component change — *"`Tier` was set to Platinum on entity 42 at 10:42"*, *"`SoftDeleted` was added to entity 87 at 11:15"*. The current state of the world is the *fold* (see [Chapter 26](./26_FiltersJoinsGroupings.md) — `Aggregate`) of all the events. Snapshots are taken periodically so the fold does not have to start from the beginning of time.

**The strengths.** This is the natural substrate for the *determinism, replay, and time-travel* properties that [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md) listed as an ECS strong suit. *"What did the world look like at 10:42 last Tuesday?"* is answerable by folding the events up to that timestamp. The full history of every entity is, by construction, retained. Audit is not a feature you add; it is the storage model.

**The cost.** Considerable. Event sourcing is a serious architectural commitment with its own substantial literature, its own failure modes (schema evolution of events, snapshot management, projection rebuilding), and its own learning curve. It is the right answer for systems whose *requirements* include replay or full history; it is dramatic overkill for systems that merely *would not mind* having them.

## How the earlier patterns snap in

The pleasing thing about persisting an ECS world is how much of Part IV applies directly.

Each component type's table or JSON shape is configured with an `IEntityTypeConfiguration<T>` — the pattern of [Chapter 29](./29_DbContextConfiguration.md) applies unchanged. A component is, after all, just a small record, and configuring its mapping is exactly the same work as configuring a small entity's mapping.

The entity ID is a perfect fit for the two-identifier pattern of [Chapter 13](../02_TheModelQuestion/13_OnIdentifiers.md). The internal integer `EntityId` does the joining between component tables — fast, compact, indexed. A public `Guid` identifies the entity to the outside world. The pattern was practically designed for this.

The metadata components of [Chapter 18](../03_ADifferentWay/18_MetadataAsAComponent.md) round-trip through exactly the same machinery as every other component. `Audit`, `SoftDeleted`, `Tenant`, `SchemaVersion` — each is a component, so each is a table (Pattern One) or a JSON entry (Pattern Two) or an event stream (Pattern Three), persisted identically to `Name` and `Tier`. There is no special metadata-persistence code, because in ECS there is no special metadata.

And the value objects of [Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md), with their value converters from Chapter 29, work inside components exactly as they work inside model entities. A component carrying a `Money` is configured with the same converter a model entity would have used.

## The concessions to EF Core

I would be misleading you if I implied this was friction-free. A few deliberate concessions make the arrangement pleasant rather than painful.

**Components used as EF entities want parameterless construction or careful configuration.** EF Core materialises rows by a mechanism that prefers either a parameterless constructor or a constructor whose parameters match the column names. A `readonly record struct Name(string Value)` works (EF Core matches `Value`), but more elaborate components may need a configured constructor binding. This is a small, one-time cost per component type.

**The in-memory ECS world and the EF Core representation are two different things.** You do not run your systems directly against the database. You load the relevant slice of the world into memory (via queries that project the components you need), run your systems against the in-memory world, and persist the changes back. This *load-modify-save* cycle is the same unit-of-work pattern EF Core already uses; it just operates on components rather than on whole entities.

**Transactions are yours to manage.** Chapter 16 flagged this as something ECS does not give you for free, and it remains true. The persistence boundary — *"these component changes commit together or not at all"* — is a transaction you open and commit explicitly, around the save phase of the load-modify-save cycle. EF Core's `SaveChanges` within a transaction scope does the actual work; the discipline of defining the boundary is yours.

## Performance, in passing

The performance characteristics differ sharply by pattern.

Pattern One (table per component) reads slowly when you want whole entities and quickly when you want single components — which, with disciplined projection, is most of the time. Writes are cheap. Reporting is excellent.

Pattern Two (JSON bags) reads whole entities quickly and individual components more slowly. Writes are cheap. Reporting is weaker, dependent on JSON-path indexing.

Pattern Three (event-sourced) writes extremely quickly (append-only), reads via projections that are themselves maintained incrementally, and offers replay at the cost of the fold. The performance profile is unusual and should be chosen deliberately, not by accident.

The headline, consistent with the whole series: *project what you need*. The ECS persistence patterns reward the same discipline as the relational ones — ask the database for the components the current operation actually requires, not the whole entity. Do that, and all three patterns perform well within their design envelope.

> *Loading an entire ECS entity's twelve components to answer a question about one of them is the persistence equivalent of unpacking the entire removal van to find the kettle — technically thorough, deeply unnecessary, and the reason the tea is going to be late.*

## The smells

- Reconstructing the full entity (all components) for an operation that touches one component.
- A JSON component bag (Pattern Two) being filtered on a JSON path that has no supporting index.
- Event sourcing (Pattern Three) adopted because it sounded sophisticated, in a system with no replay or history requirement.
- Running systems directly against the database rather than against an in-memory loaded slice.
- A component persistence path that special-cases metadata components — they are just components; persist them the same way.
- A save phase with no explicit transaction boundary — partial component writes are a corruption waiting to happen.

## Recap

- ECS breaks the ORM's *entity = class = table* assumption, so persistence needs a deliberate choice.
- Three patterns: table-per-component (relationally pure), JSON/owned-type bags (fewer joins, opaque to tooling), and event-sourced (replay and history, at significant cost).
- The patterns from Part IV apply directly — `IEntityTypeConfiguration<T>`, the two-identifier pattern, value converters — and metadata components persist identically to all others.
- The concessions: configured construction, the load-modify-save cycle, and explicit transaction boundaries.
- Project the components you need. The discipline that served relational data serves component data too.

## Onwards

That closes Part IV. We have travelled from collections, through set theory, through LINQ and EF Core, through relationships and configuration, and out the far side into the persistence of the component-shaped world that Part III argued for. Part V steps up a level again — to the architecture of the application as a whole: dependency injection without the bloat, middleware demystified, the services-versus-utilities distinction that eliminates a surprising amount of dithering, and the art of composing it all without a UML diagram on the wall.
