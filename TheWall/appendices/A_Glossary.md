# Appendix A — Glossary

Every term the series introduces, defined once and self-containedly, with a pointer to the chapter where it is properly treated. Entries are written to stand alone, so any one of them can be lifted out and read without the surrounding chapter.

---

**Anaemic model.** A class that holds data with little or no behaviour, the behaviour living instead in separate service classes. Neither inherently wrong nor inherently right — see the debate in [Chapter 11](../02_TheModelQuestion/11_RichVsAnemic.md).

**Archetype storage.** An ECS storage strategy in which entities are grouped by their exact set of component types, with each group holding parallel arrays of component data. Fast for multi-component queries; moving an entity between archetypes costs a copy. [Chapter 19](../03_ADifferentWay/19_Registries.md).

**Array of structures (AoS).** A memory layout where an array holds whole objects, each containing all its fields together. Contrast *structure of arrays*. [Chapter 19](../03_ADifferentWay/19_Registries.md).

**Boolean blindness.** The loss of meaning that occurs when a behaviour-selecting `true`/`false` reaches a call site as a bare positional argument — `Export(report, true, false)`. Cured with enums, separate methods, or an options record. [Chapter 37](../06_CodeThatDoesntSmell/37_Branching.md).

**Bounded context.** From Domain-Driven Design: the recognition that one concept (a "user") may legitimately be modelled differently in different parts of a large system, and that forcing a single shared model produces bloat. [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md).

**Boxing.** Wrapping a value type in a heap allocation so it can be treated as an `object` or interface. Silent, common, and the chief way the performance benefit of a struct is accidentally discarded. [Chapter 3](../01_TheToolsAtHand/03_Structs.md).

**Captive dependency.** A bug in which a shorter-lived service (scoped or transient) is captured by a longer-lived one (singleton), causing it to outlive its intended scope. [Chapter 31](../05_Architecture/31_DependencyInjection.md).

**Cardinality.** The numeric character of a relationship between entities: one-to-one, one-to-many, many-to-one, or many-to-many. [Chapter 28](../04_DataAndSets/28_Cardinality.md).

**Cartesian product.** The set of all pairings of one element from each of two sets; the conceptual basis of a SQL `JOIN` (a Cartesian product followed by a filter). [Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md).

**Characterisation test.** A test that pins down what a system *currently does* (bugs included), so that a subsequent refactor can be verified not to have changed observable behaviour. The safety net for refactoring legacy code. [Appendix E](./E_RefactoringOrder.md).

**Component.** In ECS, a small, flat, behaviour-free piece of data attached to an entity — near enough a `record`. The set of components an entity carries defines what it is. [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md).

**Composition root.** The single place, near the application's entry point, where the dependency-injection container is configured and the concrete implementations are wired to their interfaces. [Chapter 31](../05_Architecture/31_DependencyInjection.md).

**Covariance / contravariance (variance).** The rules (`out` / `in` on a generic type parameter) that determine when an `IEnumerable<Dog>` may be used where an `IEnumerable<Animal>` is expected, and vice versa. [Chapter 6](../01_TheToolsAtHand/06_Generics.md).

**Cross-cutting concern.** A concern that applies across many parts of the system rather than to one — logging, auditing, soft delete, tenancy. The subset-of-entities variety is the kind mainstream OO handles awkwardly and ECS handles natively. [Chapter 15](../02_TheModelQuestion/15_TheWall.md), [Chapter 18](../03_ADifferentWay/18_MetadataAsAComponent.md).

**Deferred execution.** The property of `IQueryable<T>` and lazy `IEnumerable<T>` that the query is a *recipe* not run until something materialises it — and re-run each time it is materialised. [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md).

**Dependency injection (DI).** The practice of a class being *given* its collaborators rather than constructing them itself. [Chapter 31](../05_Architecture/31_DependencyInjection.md).

**Discriminated union (DU).** A type whose value is exactly one of a closed, named set of variants, each optionally carrying its own data, with compiler-checked exhaustive handling on the consumer side. [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md).

**Domain model.** A class (or set of classes) representing a thing in the business domain together with its rules and invariants — the model in the original object-oriented sense. [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md).

**DTO (data transfer object).** A class shaped for a wire format — a JSON body, a queue message. Exists for the wire, not the domain. [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md).

**Entity (ECS).** A unique identifier — usually an integer or `Guid` — that exists only to be the thing components attach to. It has no fields, methods, or shape of its own. [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md).

**Entity Component System (ECS).** An architecture in which entities are IDs, components are flat data, and systems are stateless behaviour over entities carrying particular component sets. [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md).

**Feature envy.** A method more interested in another class's data than its own; cured by moving it to the data. [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).

**Fold (aggregate).** The reduction of a sequence to a single value by repeatedly applying a combining function; the basis of `Sum`, `Count`, `Aggregate`. The code form of a mathematical *series*. [Chapter 26](../04_DataAndSets/26_FiltersJoinsGroupings.md), [Appendix D](./D_SequencesAndSeries.md).

**Frozen collection.** A read-only collection (`FrozenDictionary<TKey,TValue>`, `FrozenSet<T>`) built once and aggressively optimised for repeated lookup. [Chapter 23](../04_DataAndSets/23_Collections.md).

**Generic.** A type or method parameterised by one or more type *parameters*, settled at the call site and checked by the compiler, with no boxing of value types. [Chapter 6](../01_TheToolsAtHand/06_Generics.md).

**God class / god entity.** A class that has accreted too many responsibilities — hundreds of lines, a name that is a category rather than a thing. [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).

**Guard clause.** An early-returning check at the top of a method that handles an exceptional case and bails out, keeping the main body at the lowest level of nesting. [Chapter 35](../06_CodeThatDoesntSmell/35_EarlyReturns.md).

**Intersection.** The set of elements present in both of two sets; the LINQ `Intersect`. [Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md).

**`IQueryable<T>` vs `IEnumerable<T>`.** The former builds an expression tree translated to a database query; the latter is an in-memory sequence evaluated in C#. Crossing from the first to the second (via `ToList()`, etc.) moves all subsequent work into memory. [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md).

**Lazy evaluation.** Computing a sequence's elements only as they are demanded, enabling infinite sequences and unmaterialised pipelines. [Appendix D](./D_SequencesAndSeries.md).

**Lifetime (service).** How long the DI container keeps a service instance: *singleton* (whole application), *scoped* (per request), *transient* (per resolution). [Chapter 31](../05_Architecture/31_DependencyInjection.md).

**Map (projection).** Transforming each element of a set or sequence into a derived value; the LINQ `Select`, the SQL `SELECT` clause. [Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md).

**Middleware.** A function that takes a request, optionally acts, and optionally calls the next function in the pipeline; the pipeline is a Russian doll whose registration order is its architecture. [Chapter 32](../05_Architecture/32_Middleware.md).

**N+1 problem.** A performance fault in which a query returning N rows triggers one further query per row, for N+1 queries where one would have done. [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md).

**Owned type.** A value object whose fields EF Core maps into its parent entity's table rather than a separate table; it has no independent identity. [Chapter 28](../04_DataAndSets/28_Cardinality.md).

**Persistence model.** A class shaped to fit a database row, with keys, foreign keys, and navigation properties. Exists for the database. [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md).

**Primitive obsession.** Representing domain values with bare `string`, `int`, or `decimal` rather than purpose-built value objects — the most preventable class of bug in business software. [Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md).

**Projection before materialisation.** The EF Core discipline of selecting the exact shape needed *before* calling `ToList()`, so the database returns only the required columns. [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md).

**PublicId.** An opaque, non-enumerable, externally-stable identifier exposed to the world, distinct from the internal integer `Id` used for joins. [Chapter 13](../02_TheModelQuestion/13_OnIdentifiers.md).

**Record.** A reference type (or, as `record struct`, a value type) with compiler-generated value equality, immutability, deconstruction, and `with`-expressions. Built for data. [Chapter 2](../01_TheToolsAtHand/02_Records.md).

**`ref struct`.** A stricter struct that may never be heap-allocated, boxed, or outlive its stack frame; the machinery behind `Span<T>`. [Chapter 3](../01_TheToolsAtHand/03_Structs.md).

**Registry / world.** In ECS, the container that owns all entities and stores their components — the moral equivalent of EF Core's `DbContext`. [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md), [Chapter 19](../03_ADifferentWay/19_Registries.md).

**Result type.** A return value representing success-or-failure explicitly (a discriminated union of `Ok`/`Err`), used in place of throwing exceptions for expected failures. [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md), [Chapter 36](../06_CodeThatDoesntSmell/36_TryCatch.md).

**Sequence.** An *ordered* collection permitting duplicates, indexed by position; the mathematical object behind `IEnumerable<T>`, `List<T>`, and `T[]`. [Appendix D](./D_SequencesAndSeries.md).

**Series.** The accumulation (typically the sum) of the terms of a sequence; the code form is a *fold*. Requires a finite sequence to terminate. [Appendix D](./D_SequencesAndSeries.md).

**Service.** A class that does work using collaborators that vary or that observes the outside world (clock, network, filesystem); belongs in the DI container. Contrast *utility*. [Chapter 33](../05_Architecture/33_ServicesVsUtilities.md).

**Set.** A collection of distinct things in no particular order; the mathematical object behind `HashSet<T>` and the basis of all querying. [Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md).

**Shotgun surgery.** A change-disease in which one logical change requires editing many files; the symptom of a concept scattered across the codebase. [Chapter 38](../06_CodeThatDoesntSmell/38_SmellsBestiary.md).

**Sparse set.** An ECS storage strategy using a dense data array plus a sparse index keyed by entity ID; simple, fast for single-component queries. [Chapter 19](../03_ADifferentWay/19_Registries.md).

**Speculative generality.** Abstraction built for an imagined future requirement that has not arrived; cured by deleting it until the second real case appears. [Chapter 38](../06_CodeThatDoesntSmell/38_SmellsBestiary.md).

**Strangler fig.** A migration strategy that grows a new system around an old one, routing new work through the new implementation and migrating existing functionality piece by piece, until the old system can be deleted. The alternative to the big rewrite. [Appendix E](./E_RefactoringOrder.md).

**Structure of arrays (SoA).** A memory layout with one array per field, indices corresponding across arrays; the ECS-natural, cache-friendly layout. Contrast *array of structures*. [Chapter 19](../03_ADifferentWay/19_Registries.md).

**Struct.** A value type, allocated (mostly) on the stack, copied by value; right for small, immutable, value-like data. [Chapter 3](../01_TheToolsAtHand/03_Structs.md).

**System (ECS).** A stateless transformation that queries the world for entities with a given component set and acts on them; behaviour lives here, not on the entity. [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md), [Chapter 17](../03_ADifferentWay/17_StatelessRevelation.md).

**Train wreck (message chain).** A line that walks a chain of properties — `a.B.C.D.E` — coupling the caller to the internal shape of a graph. [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).

**Union.** The set of elements present in either of two sets, duplicates removed; the LINQ `Union` (contrast `Concat`, which keeps duplicates). [Chapter 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md).

**Utility.** A pure function — no dependencies, no state, deterministic — whose right home is a static method, not the DI container. Contrast *service*. [Chapter 33](../05_Architecture/33_ServicesVsUtilities.md).

**Value object.** A small type whose purpose is to *be* a particular kind of value with its rules enforced at construction; the cure for primitive obsession. [Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md).

**View model.** A class shaped to fit a particular screen or component. Exists for the user interface. [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md).
