# Chapter 19 — Registries: Types All the Way Down

The previous chapters have treated the *world* as a black box. *"Add this component, query for those components"* — and the right thing happens. This chapter lifts the lid on the box and looks at the machinery underneath, because the storage and dispatch strategy of an ECS framework is one of the more interesting engineering problems in modern software, and understanding it changes the way you write the code that sits on top.

The good news is that you do not need to understand every detail of the storage to be productive in an ECS codebase, any more than you need to understand the inside of a B-tree to write good SQL. The bad news is that the leaks from this layer to the application layer are real — knowing what is cheap and what is expensive will affect how you write systems — so a chapter spent here pays back through the rest of Part III.

## What the registry actually has to do

The world's external API is, as we saw in [Chapter 16](./16_ECSFirstLook.md), small:

```csharp
world.Create();
world.Add(entity, new Position(0, 0));
world.Has<Velocity>(entity);
world.Get<Position>(entity);
world.Set<Position>(entity, newPosition);
world.Remove<Velocity>(entity);
world.Destroy(entity);
world.Query<Position, Velocity>();
```

To deliver that API, the registry needs to solve four problems.

1. **Identify components by type.** *"This is a `Position`, that is a `Velocity`"* — the framework needs a runtime handle on each component type.
2. **Find an entity's components quickly.** Given an entity ID and a component type, return the component data — and report quickly whether it exists at all.
3. **Iterate entities with a given component set quickly.** Given a query for `<Position, Velocity>`, produce every entity that has both, and ideally produce the component data along with the IDs.
4. **Handle add and remove gracefully.** Adding or removing a component changes which queries an entity matches; the storage needs to keep up without making the application code think about it.

There are two dominant strategies for solving these. Both are interesting. Both have trade-offs. Most production ECS frameworks pick one as the default and offer the other as an option.

## Approach one — sparse sets

The simpler of the two strategies is *sparse-set storage*. Each component type has its own pair of arrays.

```
PositionStorage:
  sparse:  [_, _, 0, _, 1, _, 2, ...]   indexed by entity ID
  dense:   [(e2, pos2), (e4, pos4), (e6, pos6), ...]   contiguous data
```

The *dense* array holds the component data, packed contiguously, with no gaps. The *sparse* array, indexed by entity ID, tells you the entity's position in the dense array — or that it has no component of this type.

To check `Has<Position>(entity)`, look in the sparse array; the index is valid if and only if the entity carries the component.

To `Get<Position>(entity)`, look in the sparse array, then dereference the dense array. Two memory accesses, both bounded.

To iterate `Query<Position>()`, walk the dense array directly. Cache-friendly, no gaps, no branch misprediction.

To `Add<Position>(entity, pos)`, append to the dense array and update the sparse index.

To `Remove<Position>(entity)`, swap the last element of the dense array into the freed slot and decrement the count. This is the famous *swap-remove* trick. It is O(1). The downside is that iteration order is not stable across removes, but for the kinds of work systems do, this is rarely a problem.

The sparse-set approach is the default in *EnTT* and a common choice elsewhere. It is straightforward to implement, performs well for single-component queries, and degrades gracefully as the system grows.

## Approach two — archetypes

The more sophisticated approach is *archetype storage*. An archetype is a *set of component types* — say, `(Position, Velocity)` or `(Position, Velocity, Health)`. Every entity belongs to exactly one archetype at any given moment, namely the archetype whose component set matches the entity's exactly.

```
Archetype (Position, Velocity):
  entities:    [e2, e7, e11, ...]
  Position[]:  [pos2, pos7, pos11, ...]   parallel arrays
  Velocity[]:  [vel2, vel7, vel11, ...]

Archetype (Position, Velocity, Health):
  entities:    [e1, e5, ...]
  Position[]:  [pos1, pos5, ...]
  Velocity[]:  [vel1, vel5, ...]
  Health[]:    [hp1, hp5, ...]
```

To iterate `Query<Position, Velocity>()`, walk *every archetype that contains both* `Position` *and* `Velocity` — the first one above, the second one above, and any others in the world that include both. Within each archetype, the iteration is *straight-line walks of parallel arrays* — the most cache-friendly access pattern modern hardware supports.

The cost is that adding or removing a component on an entity *changes the entity's archetype*, which requires moving its data from one archetype's arrays to another's. For workloads where entities have stable component sets, this is essentially free. For workloads that thrash components on and off entities, it can become a bottleneck.

Archetype storage is the default in *Unity DOTS*, *Bevy*, and most of the modern game-engine ECS implementations. It is more elaborate than the sparse-set approach, but it pays back handsomely on multi-component queries — which, in a mature codebase, is most queries.

## Identifying components by type

Both approaches need a way to map a `Component` type to a stable integer ID. C#'s generic system makes this pleasingly easy.

```csharp
public static class ComponentTypeId<T>
{
    public static readonly int Value = ComponentRegistry.Register(typeof(T));
}
```

`ComponentTypeId<Position>.Value` is computed exactly once, the first time the static field is accessed, and is the same integer forever after. Looking up *"which storage holds `Position` components?"* becomes a single array index. The runtime never needs to compute a hash, walk a dictionary, or call `typeof()` repeatedly in a hot loop.

This is a small trick, and it is one of the reasons C# generics turn out to be such a good substrate for ECS — see [Chapter 6](../01_TheToolsAtHand/06_Generics.md) for the broader argument. The type system gives the framework a free per-type ID at the cost of a static class definition. Languages without this property (Python, JavaScript) have to fall back to runtime dictionaries keyed on strings or class objects, with the corresponding slowdown.

## Memory layout, briefly

You will, when reading about ECS, encounter the phrase *structure-of-arrays* (SoA) contrasted with *array-of-structures* (AoS). It is worth a short paragraph.

An *array of structures* is the OO-natural layout: an array of objects, each of which contains all of an entity's data. To read just the `Position` of every entity, the CPU has to walk the whole array touching every byte of every object.

A *structure of arrays* is the ECS-natural layout: one array per component type, with corresponding indices belonging to the same entity. To read just the `Position` of every entity, the CPU walks one tight contiguous array of `Position`s and touches nothing else.

For most business code, this distinction is unmeasurable. For workloads that process large numbers of entities — bulk recalculations, scheduled reports, batch jobs — the SoA layout can be several times faster, sometimes an order of magnitude faster, with no algorithmic change.

> *The difference between AoS and SoA, for the right workload, is rather like the difference between reading every book on a shelf cover-to-cover to find every mention of Tuesday, and reading just the index. Both will work; only one will leave you with time to also have lunch.*

## Dispatch — making `Query<Position, Velocity>` fast

The third hard problem is dispatching the query itself. Naively, `Query<T1, T2>()` could walk every entity in the world, check whether each has both `T1` and `T2`, and yield matches. This is correct and unforgivably slow.

The sparse-set approach optimises this by *intersecting* the dense arrays of the participating component types — walk the smaller one, and for each entity in it, check the sparse arrays of the other component types. The intersection cost is bounded by the smallest dense array, which is usually a very large speedup over walking every entity.

The archetype approach optimises differently — *enumerate the archetypes that contain all the requested component types* (a small, mostly static list) and walk each one straight through. There is no per-entity check; every entity in a matching archetype is, by construction, a match. This is the fastest dispatch any approach has yet produced for multi-component queries, and it is why the modern game engines have gone this route.

## Source generators — the modern C# advantage

A delightful property of modern C# that ECS frameworks can lean on heavily is *source generators*. Rather than registering component types and systems at runtime via reflection — which works but has a startup cost and a small per-operation cost — the framework can emit, at compile time, the exact dispatch code for every query in the application. The result is a query that looks like:

```csharp
foreach (var (entity, pos, vel) in world.Query<Position, Velocity>())
{
    // body
}
```

…and compiles down to something approximately as fast as the hand-written equivalent, with no per-call overhead, no boxing, no generic-dispatch tax.

This is also where C# pulls comfortably ahead of the older sparse-set frameworks in C++, which rely on template metaprogramming for the same effect at considerably greater syntactic cost. The C# story for ECS, from the mid-2020s onwards, is genuinely competitive with anything the field has produced.

## Performance, in passing

To put numbers on the abstract claims of the previous chapters: a well-tuned archetype-based ECS framework on modern hardware can iterate a multi-component query at *hundreds of millions of entities per second per core*. This is not a number that matters for a typical business application, but it is worth knowing the headroom exists. The performance of well-designed ECS code is essentially never the bottleneck.

The bottleneck in an ECS-backed business application is almost always either I/O (the database, the network) or the structural quality of the systems themselves — a poorly factored system that does ten times more work than it needs to is slow whether it lives in ECS or in OO. The runtime layer rarely contributes to the slowness.

## What this means for the system author

Three small, practical implications for code you will actually write.

**Multi-component queries are cheap; iterate-then-`Has` checks are expensive.** `Query<Position, Velocity>()` is fast because the framework optimises it. `foreach (var e in world.Query<Position>()) if (world.Has<Velocity>(e)) ...` is slow because it forces a per-entity check. Reach for the multi-component query whenever you can.

**Adding and removing components is not free.** It is cheap, but it is not free, and a system that toggles a component on every entity every tick is doing more work than one that uses a persistent component plus a boolean field. As always, measure before optimising, but be aware of the cost shape.

**Component storage decisions are usually the framework's, not yours.** You will rarely need to choose between sparse-set and archetype storage at the application level; pick a framework whose default suits your workload and trust the default. If you find yourself outgrowing it, the framework will give you the knobs.

## Recap

- The registry's job is to identify components by type, find components by entity, iterate by component set, and handle adds and removes cleanly.
- The two dominant strategies are *sparse sets* (simple, fast for single-component queries) and *archetypes* (more elaborate, fastest for multi-component queries).
- C# generics give ECS frameworks free per-type integer IDs, which keep the dispatch layer fast.
- Structure-of-arrays storage is the natural fit for ECS and is what gives the pattern its cache-friendliness.
- Source generators in modern C# bring query dispatch down to near-hand-written speeds.

## Onwards

The next chapter steps from *how the runtime works internally* to *how the runtime gets configured* — the registration of components and systems without the assembly-scanning gymnastics that have, over the years, made mainstream DI containers slower, more mysterious, and harder to reason about than they should be. It is one of the quieter operational wins of an ECS architecture, and worth a chapter to itself.
