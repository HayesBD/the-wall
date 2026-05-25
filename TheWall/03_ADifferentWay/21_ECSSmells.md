# Chapter 21 — The ECS Code Smells Bestiary

The mirror chapter of [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md). Where that one catalogued the ways model-based codebases curdle, this one catalogues the ways ECS codebases do — and they do — when the discipline that the pattern depends on starts to slip.

It is worth saying up front that *every architecture has its smells*, and the existence of a smells chapter is not, by itself, an argument against the pattern. The model smells chapter did not invalidate models; it was a tool for keeping model codebases healthy. This chapter serves the same purpose for ECS codebases. Read it in that spirit.

The groupings are mine; the smells themselves I have, over the years, watched ECS codebases produce often enough that I can describe each one without needing to invent it.

## Group one — the drift-back-to-models smells

**The bloated component.** A `Customer` component with twenty fields. *Symptom:* the component definition is a page long; changes to it ripple through every system that queries it. *Cause:* a developer with model-shaped instincts has used the component as a record-like aggregate, missing the point of splitting it into smaller pieces. *Cure:* refactor into a handful of focused components — `Identity(Name)`, `CustomerTier(Level)`, `Audit(...)`, `LoyaltyPoints(Value)` — each held by entities that need it. The aggregate is not the unit of ECS design; the concern is.

**The hidden inheritance hierarchy.** Components named `BaseEntity`, `AuditableThing`, `OrderableItem`. *Symptom:* a tag-shaped component that does not carry data, used to mark *kinds* of entity rather than *capabilities*. *Cause:* an instinct to recreate the *is-a* relationships of OO inheritance in ECS clothing. *Cure:* think *has-a*, not *is-a*. Replace the marker with components that describe the actual capability — `Auditable` becomes attaching an `Audit` component, `Orderable` becomes attaching the `Stock` and `Pricing` components that make ordering meaningful. The capability is the contract; the marker is theatre.

**The aggregate that wants to be an entity.** A system that takes one entity ID and walks a graph of related entities to do its work. *Symptom:* `world.GetRelated<Order>(customer)`-shaped methods that hide entity-to-entity navigation. *Cause:* missing the ECS framing — relationships between entities should be components on one or the other, not invisible graph walks. *Cure:* model relationships as components carrying entity IDs (`OwnedBy(EntityId Owner)`), or as separate relationship-entities for many-to-many links. Make the structure visible.

## Group two — the hidden-state smells

**The stateful system.** A system class with mutable fields — a cache, a list, a counter, an event subscription. *Symptom:* `private readonly Dictionary<...> _cache` on a system. *Cause:* the developer instinct to optimise by holding state, or to coordinate across system invocations by hiding bookkeeping in fields. *Cure:* move the state into the world as a component (on a singleton entity, if it is global) or into a dedicated *resource* if the framework supports them. The discipline that systems hold no state (see [Chapter 17](./17_StatelessRevelation.md)) is what gives ECS its testability, parallelism, and replay; the moment a system has a field, all three properties begin to leak.

**The cache that thinks it is helping.** A system that, *"for performance"*, maintains a local cache of computed values. *Symptom:* a `_computedTiers` dictionary inside `LoyaltyPromotionSystem`. *Cause:* an attempt to optimise across system runs without using the world. *Cure:* if the cached value is per-entity, it is a component (`ComputedTier(Level)`); if it is global, it is a singleton component. Either way, it lives in the world, is queryable, is serialisable, and survives the next reload — none of which is true of the field-based cache.

**The mutation that bypasses the world.** Code that does `var pos = world.Get<Position>(entity); pos.X = 10;` — reaching into the component and mutating it directly. *Symptom:* component changes that do not trigger change-detection, do not produce events, do not appear in replay logs. *Cause:* components are records; records support property setters if the developer wrote them; the developer wrote them. *Cure:* make components `readonly` (and prefer `readonly record struct`), forcing all updates to go through `world.Set<T>(...)`. The compiler then enforces the discipline the framework relies on.

## Group three — the over-componentisation smells

**The component explosion.** Fifty components where five would have done. *Symptom:* a trivial change to a customer touches `CustomerName`, `CustomerEmail`, `CustomerPhone`, `CustomerAddressLine1`, `CustomerAddressLine2`, `CustomerPostcode` — each its own component, each its own system access pattern. *Cause:* misreading the rule *"components should be small"* as *"components should be tiny"*. *Cure:* group fields that are *always read together* into the same component. `ContactDetails(Email, Phone)` is one component, not two. The unit of componentisation is the *concern*, not the *field*.

**The systems-for-every-method.** Eighty systems where the same job, conceptually, would have been one. *Symptom:* `CustomerCreateSystem`, `CustomerUpdateSystem`, `CustomerDeleteSystem`, `CustomerValidateSystem`, `CustomerEnrichSystem`, each invoked sequentially for what is really *"handle a customer write"*. *Cause:* a misreading of *"systems should be focused"* as *"systems should be single-method"*. *Cure:* a system represents one *concern*, not one *operation*. `CustomerWritePipeline` may legitimately handle several related operations if they share a query and a transaction boundary.

**The marker-component soup.** Every entity decorated with five tag components — `IsCustomer`, `IsActive`, `HasOrders`, `IsPremium`, `IsTrial` — used to drive query filtering. *Symptom:* the smallest query joins four sparse-set lookups before it touches any actual data. *Cause:* trying to encode *"kind of thing"* with markers rather than with data. *Cure:* most of these markers are better expressed as properties of a single component — `Status(Active, Trial, Premium)` rather than three separate marker components — or as the presence/absence of data components that already carry the meaning.

## Group four — the coordination smells

**The implicit system ordering.** Two systems that *must* run in a particular order or the world ends up in an invalid state, with no declaration anywhere of the dependency. *Symptom:* a bug that appears intermittently after someone reorders the `SystemRegistration` block. *Cause:* the systems are communicating through shared component state without making the dependency explicit. *Cure:* declare the ordering in the framework's scheduler (most ECS frameworks support `RunAfter`/`RunBefore` or system groups), or — better — refactor so the systems either commute (order does not matter) or communicate through events whose causal order is explicit.

**The cross-system method call.** A system that takes another system as a constructor parameter and calls into it directly. *Symptom:* `LoyaltyPromotionSystem(IAuditSystem audit)` and `audit.Record(...)` calls inside the loop. *Cause:* importing OO-shaped collaboration patterns where they do not belong. *Cure:* systems communicate by mutating the world. If `LoyaltyPromotionSystem` wants `AuditSystem` to notice a change, it sets a component (`PendingAudit`); the `AuditSystem`, on its next run, processes everything carrying `PendingAudit`. The coupling moves from compile-time to runtime, where ECS can manage it.

**The query inside a query.** A system that, inside `foreach (var (e, pos) in world.Query<Position>())`, calls `world.Query<Velocity>()` again, producing the ECS equivalent of an N+1 problem. *Symptom:* a system whose runtime grows quadratically in the number of entities. *Cause:* the developer needed components from both queries and reached for nested iteration. *Cure:* `world.Query<Position, Velocity>()` — the multi-component query the framework optimises (see [Chapter 19](./19_Registries.md)). One walk. One result.

## Group five — the lifecycle smells

**The orphan entity.** Entities created during a run and never destroyed. *Symptom:* the world grows in size monotonically; queries get slower; memory usage climbs without bound. *Cause:* missing the *destroy* step in some lifecycle path — usually an error case that returns early without cleanup. *Cure:* either own the lifecycle explicitly (every `Create` paired with an eventual `Destroy`), or use a *generational* entity ID where stale references are detected automatically. Most modern ECS frameworks support the latter.

**The component leak.** Components added during an operation and never removed, even though the entity is supposed to have returned to a clean state. *Symptom:* entities accumulate transient components (`Processing`, `PendingValidation`, `WaitingForApproval`) that the systems responsible for clearing them have stopped running. *Cure:* the system that *adds* a transient component owns the responsibility to *remove* it. Treat transient components as a small lifecycle contract.

**The destroyed-entity reference.** Code that holds an `EntityId` past the destruction of the entity it referred to. *Symptom:* a `world.Get<Position>(staleId)` that throws or returns garbage. *Cause:* the same lifecycle hazard as a dangling pointer, in slightly more polite clothing. *Cure:* either the framework's generational IDs (which return a clear *"that entity is gone"* result), or the discipline of not storing entity IDs across system invocations without re-validating them.

## A small word on discipline

ECS is a pattern that *rewards discipline more than it tolerates indiscipline*. A model-based codebase with mediocre discipline limps along for years before the wall arrives. An ECS codebase with mediocre discipline goes wrong faster — the bloated components and hidden-state systems and over-componentisation can together turn an ECS codebase into a worse mess than the model approach it was meant to replace, and can do so in months.

This is not, I want to say, a reason to avoid ECS. It is a reason to take its disciplines seriously when you adopt it. The model approach got fifty years of accumulated wisdom about how to do it well — *Refactoring*, the *Gang of Four*, *Domain-Driven Design*, the long literature of code reviews and team conventions. The ECS approach is younger, less widely understood, and has, in mainstream business software, much less of that wisdom written down. A team adopting ECS has to bring its own.

> *Adopting ECS without the discipline to keep components small, systems stateless, and lifecycles clean is rather like buying a racing bicycle and never quite getting around to oiling the chain: the equipment is capable of remarkable things, and you have very deliberately denied yourself any of them.*

## Performance, in passing

A respectable share of these smells have measurable performance costs as well as readability ones. The query-inside-a-query is the headline example — quadratic instead of linear. The component explosion adds per-query lookups. The orphan entities accumulate cache pressure. The stateful systems prevent parallelism. Each is, in addition to being aesthetically displeasing, *slow*.

The good news is that the same fixes that restore the architecture's clarity tend to restore its performance. Treating the smells as design problems rather than performance problems is the right framing; the performance recovery is, again, the dividend.

## Recap

- ECS has its own catalogue of smells, mostly clustered around drift-back-to-models, hidden state, over-componentisation, implicit coordination, and lifecycle hazards.
- The pattern rewards discipline and punishes indiscipline more sharply than the model approach does.
- Most of the smells have direct cures rooted in the principles of [Chapter 17](./17_StatelessRevelation.md): components are flat data, systems are stateless, communication goes through the world.
- The performance cost of the smells, as with the model bestiary, is usually a shadow cast by the structural problem.

## Onwards

Both architectures have now had their fair hearing, their best case made, their failure modes catalogued. The next chapter — [Chapter 22](./22_ModelsVsECS.md) — puts them side by side, with the same problem expressed both ways, and hands the verdict back to the reader. I have tried, through Part II and Part III, to make the case for each as honestly as I can manage. What the reader does with the comparison is up to them.
