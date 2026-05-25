# The Wall
### *A Reasonably Opinionated Field Guide to Writing Code That Doesn't Embarrass You Later*

---

## Foreword
A short, slightly indulgent piece on why this book exists and who it is for. Sets the contract with the reader: these are the opinions I've arrived at after a great deal of code; they are held with confidence and equally with curiosity. If you've found something cleaner, I want to hear about it.

---

## Part I — The Tools at Hand
*In which we take inventory of the C# type system and the concurrency machinery, and discover, to our mild surprise, that we have rather a lot of furniture to choose from.*

- **Chapter 1 — A Brief Tour of the Toolshed**
  Classes, structs, records, enums, interfaces, and the difference between *having* a hammer and *needing* one.

- **Chapter 2 — Records: The Quiet Revolution**
  Immutability without ceremony. When records are the right answer and when they are the answer to a question nobody asked.

- **Chapter 3 — Structs and the Stack**
  A short, careful chapter on value types, allocations, and why "premature optimisation" is not a licence to ignore them entirely.

- **Chapter 4 — Enums, and Their Many Sins**
  Enums as labels, enums as flags, enums masquerading as states, and the three reasons enums are usually a code smell in disguise.

- **Chapter 5 — Discriminated Unions: The Long-Awaited Guest**
  An honest look at the preview-feature DUs arriving in modern C#, with a clearly marked sidebar on the fallback patterns for those still on stable releases.

- **Chapter 6 — Generics, and a Word on the Competition**
  Why generics are one of the things C# gets gloriously right. *(Contains a polite raised eyebrow in the direction of Python and JavaScript — they don't have this, and the absence shows.)*

- **Chapter 7 — Blueprints vs Prototypes: How Objects Get Made**
  Class-based instantiation as we know it in C#. JavaScript's prototype chain and the curious things it permits. Python's "four pillars but virtually" — object orientation by polite agreement rather than by enforcement. Why understanding the differences makes you a better C# developer even if you never write a line of either.

- **Chapter 8 — Async, Await, and the Task Machine**
  What `async`/`await` actually compiles to, what a `Task` actually is, and why most of the bugs people attribute to "async being hard" are really bugs of misunderstanding the state machine.

- **Chapter 9 — Threading: When the Task Isn't Enough**
  When `Task.Run` is the right reach and when it is a cargo cult. Threads, the thread pool, parallelism vs concurrency, `Parallel.ForEach`, `Channel<T>`, and the small handful of cases where you genuinely need to think about a thread by name.

---

## Part II — The Model Question
*In which we examine that most cherished of object-oriented institutions, give it a fair hearing, and quietly note where the seams begin to fray.*

- **Chapter 10 — What Is a Model, Really?**
  Strip back the assumptions. A model is not a database row. A model is not a DTO. A model is not, despite popular belief, whatever EF Core gave you last Tuesday.

- **Chapter 11 — Rich vs Anemic: The Eternal Debate**
  Both camps presented at their best. The case for behaviour-on-the-model. The case for behaviour-elsewhere. Where each thrives and where each curdles.

- **Chapter 12 — Primitive Obsession and Its Discontents**
  The single most under-discussed source of bugs in business software. Why `string customerId` is a tiny treachery, and how to retire it without inventing a thousand value-object classes.

- **Chapter 13 — On Identifiers: `Id`, `PublicId`, and the Trouble with Exposing Either**
  Why the integer primary key should never leave the building. Public IDs, opaque identifiers, model binding in the DbContext, and the elegant pattern that makes the whole thing disappear from your controllers. *(The hookup itself is built in Chapter 29.)*

- **Chapter 14 — The Model Code Smells Bestiary**
  A field guide to the ten or so recurring offences: anaemic getters, boolean parameters, god classes, train wrecks, primitive parties, and the dreaded `Manager` suffix.

- **Chapter 15 — The Wall: When Models Stop Scaling**
  An honest chapter. The codebase that worked beautifully at twenty entities and now, at two hundred, has begun to creak. What actually slows feature delivery — and it is rarely what the standup blames.

---

## Part III — A Different Way of Thinking
*In which we set the models aside, briefly, and consider an architecture that grew up in game engines and is, perhaps, more applicable to the rest of us than we have been led to believe.*

- **Chapter 16 — Entities, Components, Systems: A First Look**
  The mental model, explained without jargon. An entity is an ID. A component is data. A system is behaviour. That's the whole magic trick.

- **Chapter 17 — The Stateless Revelation**
  Why components are flat, dumb, and brilliant. Why systems are stateless. Why this combination, which feels initially like a betrayal of every OO instinct, turns out to be liberating.

- **Chapter 18 — Metadata as a Component: The Quiet Superpower**
  Created-at, created-by, schema version, tags, soft-delete flags, dirty markers — none of these need a special parking spot any more. They are components, queryable like any other, processed by whichever systems care. Introspection becomes free, debugging becomes a query, and the cross-cutting concerns that haunted the base class melt away.

- **Chapter 19 — Registries: Types All the Way Down**
  Component types, entity types, system types, assembly registries. The plumbing that makes runtime composition possible without sacrificing the compiler's help.

- **Chapter 20 — Bindings Without Reflection-by-Assembly**
  How ECS sidesteps the assembly-scanning gymnastics that haunt mature DI-heavy codebases. More work up front. Considerably less work at the eighteen-month mark.

- **Chapter 21 — The ECS Code Smells Bestiary**
  Yes, ECS has its own. Bloated components, systems that secretly hold state, registries that become god objects, and the temptation to recreate inheritance through composition.

- **Chapter 22 — Models vs ECS: The Reckoning**
  Side by side. Same problem, both approaches. Strengths, weaknesses, costs, payoffs. The verdict is left, scrupulously and entirely, to the reader.

---

## Part IV — Data, Collections, and the Mathematics Hiding Underneath
*In which we discover that the things we have been doing to data for years have, all along, been a branch of mathematics with a perfectly good name for itself — and that the database is closer to friend than foe.*

- **Chapter 23 — Collections: What Most People Get Wrong**
  Arrays, lists, hash sets, dictionaries, queues, stacks. A clear-eyed look at which to reach for and when. The cost of choosing wrong.

- **Chapter 24 — A Short Polemic on Weakly Typed Arrays**
  Four small, well-evidenced observations on what other languages call "arrays" and why C# developers should appreciate, just for a moment, what they have.

- **Chapter 25 — Set Theory for Five-Year-Olds**
  No, really. The set of all dogs. The set of all spaniels. The set of all spaniels who are also good boys. By the end of this chapter, a child could follow it. By the end of the next, that child could write LINQ.

- **Chapter 26 — Filters, Joins, Groupings: It Was Sets All Along**
  Every query you have ever written. Every `WHERE`, `JOIN`, `GROUP BY`, every LINQ `Where`, `SelectMany`, `GroupBy` — all of it is set theory wearing a high-vis vest. Once you see it, you can't unsee it.

- **Chapter 27 — LINQ and EF Core: Writing Queries That Don't Make DBAs Cry**
  Deferred execution, the great IQueryable/IEnumerable schism, the `N+1` problem, projection before materialisation, and the small rituals that turn a 4-second page load into a 40-millisecond one.

- **Chapter 28 — Cardinality and Relationships: Modelling the Shape of the World**
  One-to-one, one-to-many, many-to-many — the four cardinalities and the honest question each is answering. Where developers reach for the wrong shape and pay for it later. Many-to-many with and without an explicit join entity, and why the explicit join almost always wins the moment the relationship grows an attribute of its own. Required vs optional ends. Navigation properties on one side or both. Owned types. The relationship smells nobody warned you about.

- **Chapter 29 — The DbContext Configuration Pattern: Where Persistence Concerns Belong**
  Data annotations vs fluent `OnModelCreating` vs `IEntityTypeConfiguration<T>` — and why the per-entity config class is the one to reach for the moment the model grows past a handful of types. `ApplyConfigurationsFromAssembly`, the model-binding hookup promised in Ch. 13, value converters for primitive-killing value objects, query filters, owned-type configuration, and the unwavering discipline of keeping the domain model unaware that it lives in a database.

- **Chapter 30 — Persisting an ECS World: Where Components Meet Relational Tables**
  Three honest patterns for saving an ECS world to a relational database: table-per-component, owned-type/JSON-column component bags, and event-sourced component changes. How `IEntityTypeConfiguration<T>` (Ch. 29) maps cleanly to a component, how the public-id pattern (Ch. 13) becomes the natural entity identifier, and how metadata components (Ch. 18) round-trip unchanged. The small concessions to EF Core that make the whole arrangement pleasant rather than painful.

---

## Part V — Application Architecture Without the Theatre
*In which we tackle the architectural decisions that are, in honest practice, where most teams either thrive or quietly suffocate.*

- **Chapter 31 — Dependency Injection, and How to Stop It Eating the House**
  DI is wonderful. DI containers with three hundred registrations are not. How to keep the container useful, the constructors short, and the test fixtures sane.

- **Chapter 32 — Middleware: What It Actually Is**
  A surprising number of working developers cannot, if pressed, define middleware without gesturing. We fix that. What it is, what it isn't, when to write one, and when the answer is *"please, anything but middleware"*.

- **Chapter 33 — Services vs Utilities: Knowing the Difference**
  The distinction that, once internalised, eliminates a remarkable amount of architectural dithering. State or no state. Lifetime or no lifetime. Inject or call.

- **Chapter 34 — The Art of Clean Composition**
  How well-named, well-scoped, well-bounded pieces fit together without needing a UML diagram on the wall.

---

## Part VI — Writing Code That Doesn't Smell
*In which we collect, label, and learn to recognise the small daily habits that separate code one is proud of from code one keeps quiet about at parties.*

- **Chapter 35 — Early Returns and Guard Clauses**
  The single highest-impact, lowest-effort change most developers can make. Why the pyramid of doom is not, in fact, mandatory.

- **Chapter 36 — Try/Catch: When To, When Absolutely Not To, and When You Should Be Ashamed**
  Exceptions are not flow control. Catching `Exception` is not a strategy. A chapter that may, in places, raise its voice slightly.

- **Chapter 37 — Branching: The Silent Killer**
  Why every `if` is a tiny tax on the reader. Polymorphism, pattern matching, lookup tables, and the strategy pattern as humane alternatives.

- **Chapter 38 — The Complete Code Smells Bestiary**
  A consolidated reference. Every smell discussed in earlier chapters, plus a dozen the series hasn't had room for, each with its symptoms, its causes, and its cure.

---

## Part VII — Putting It All Together
*In which we take the whole toolkit out for a drive and see what it can do.*

- **Chapter 39 — A Case Study**
  One problem, taken from sketch to working software, applying the principles of the series end-to-end. The domain itself is to be chosen with the reader in mind — we will work it out before drafting. *(The astute reader may notice which architecture survives the late-chapter feature requests with its dignity intact.)*

- **Chapter 40 — Onwards**
  Where to read next. What this book deliberately did not cover. A short, slightly wistful farewell.

---

## Appendices

- **Appendix A — Glossary**
  Every term introduced in the series, defined once, properly, for the record.

- **Appendix B — A Reading List**
  The books, papers, and talks that shaped the views expressed here. Honest credit where it is due.

- **Appendix C — Cheat Sheets**
  One-page summaries: the smells, the LINQ-to-SQL equivalences, the DI lifetimes, the async dos and don'ts, the EF Core configuration patterns.

- **Appendix D — Sequences, Series, and the Shapes Data Takes**
  The mathematical sequel to the set-theory chapters. Sets vs sequences vs series, and how the three map exactly onto `HashSet<T>`, `IEnumerable<T>`, and `Aggregate`. Lazy and infinite sequences, the convergence/termination caution, and the bugs that cluster where code assumed one shape and the data was another.

- **Appendix E — The Strangler's Path: Cleaning Up a Codebase in the Right Order**
  Given a large, valuable, not-good codebase, the logical order to apply everything in the book — safety net, observability, free wins, dependencies, value objects, queries, model separation, by-feature reorganisation, and ECS last and selectively. Built on the strangler-fig principle: never rewrite; always strangle.

---

## Folder Layout (Scaffolded)

```
TheWall/
├── README.md
├── TABLE_OF_CONTENTS.md          (this file)
├── 00_Foreword/
│   └── Foreword.md
├── 01_TheToolsAtHand/
│   ├── 01_AToolshed.md
│   ├── 02_Records.md
│   ├── 03_Structs.md
│   ├── 04_Enums.md
│   ├── 05_DiscriminatedUnions.md
│   ├── 06_Generics.md
│   ├── 07_BlueprintsVsPrototypes.md
│   ├── 08_AsyncAwaitTasks.md
│   └── 09_Threading.md
├── 02_TheModelQuestion/
│   ├── 10_WhatIsAModel.md
│   ├── 11_RichVsAnemic.md
│   ├── 12_PrimitiveObsession.md
│   ├── 13_OnIdentifiers.md
│   ├── 14_ModelSmells.md
│   └── 15_TheWall.md
├── 03_ADifferentWay/
│   ├── 16_ECSFirstLook.md
│   ├── 17_StatelessRevelation.md
│   ├── 18_MetadataAsAComponent.md
│   ├── 19_Registries.md
│   ├── 20_BindingsWithoutAssembly.md
│   ├── 21_ECSSmells.md
│   └── 22_ModelsVsECS.md
├── 04_DataAndSets/
│   ├── 23_Collections.md
│   ├── 24_WeaklyTypedArrays.md
│   ├── 25_SetTheoryForFiveYearOlds.md
│   ├── 26_FiltersJoinsGroupings.md
│   ├── 27_LinqAndEfCore.md
│   ├── 28_Cardinality.md
│   ├── 29_DbContextConfiguration.md
│   └── 30_PersistingECS.md
├── 05_Architecture/
│   ├── 31_DependencyInjection.md
│   ├── 32_Middleware.md
│   ├── 33_ServicesVsUtilities.md
│   └── 34_CleanComposition.md
├── 06_CodeThatDoesntSmell/
│   ├── 35_EarlyReturns.md
│   ├── 36_TryCatch.md
│   ├── 37_Branching.md
│   └── 38_SmellsBestiary.md
├── 07_TogetherNow/
│   ├── 39_CaseStudy.md
│   └── 40_Onwards.md
└── appendices/
    ├── A_Glossary.md
    ├── B_ReadingList.md
    ├── C_CheatSheets.md
    ├── D_SequencesAndSeries.md
    └── E_RefactoringOrder.md
```
