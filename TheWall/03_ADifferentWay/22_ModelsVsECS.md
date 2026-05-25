# Chapter 22 — Models vs ECS: The Reckoning

We have spent twelve chapters on this question across two Parts, and the time has come to put the two approaches side by side and look at them. I have tried, throughout, to be fair-minded — to present each at its best, with its most modern techniques, with its smells catalogued, with its proper trade-offs acknowledged. Whether I have succeeded in that fairness is a judgment I will leave to you.

What I will *not* do in this chapter is tell you which to pick. There are several reasons for that, but the principal one is that *the right answer genuinely depends on your situation*, and a chapter that says *"the right answer is X"* would be lying to a respectable fraction of its readers. What I will do, instead, is lay out the comparison as clearly as I can, surface the conditions under which each approach is the better choice, and trust you to do the rest.

## The same problem, both ways

Let me start with the running example we have used through both Parts: a `Customer` concept, accreting a few features over the course of six months.

**The starting point — both approaches at month one.**

In the model approach, a class:

```csharp
public class Customer
{
    public CustomerId Id { get; private set; }
    public string Name { get; private set; }
    public DateOnly JoinedOn { get; private set; }
}
```

In the ECS approach, an entity with components:

```csharp
var alice = world.Create();
world.Add(alice, new Name("Alice"));
world.Add(alice, new JoinedOn(new DateOnly(2025, 1, 1)));
```

Both are short. Both are clear. Both will serve the application equally well in its first month. There is no honest case for ECS over models *at this scale*; the model is the simpler answer to the simpler problem.

**Month three — add a tier.**

The model gets a `Tier` field, a `Promote` method, a constraint on the column, an update to the `IEntityTypeConfiguration<Customer>`, an update to the two DTOs that already serialise the customer, an update to the one view model that already renders them.

The ECS world gets a new `CustomerTier(TierLevel Level)` component and a `LoyaltyPromotionSystem` that watches for entities with both `CustomerTier` and `LifetimeSpend`. The existing entities continue to work; the new entities can opt in by carrying the new component.

The two are, at this stage, roughly comparable in cost. The model touches more files; the ECS approach involves a new component type and a new system class. Neither is dramatically better.

**Month six — add soft delete, tagging, suspension, audit.**

The model approach acquires four new nullable columns on `Customer` (or — if discipline holds — a `CustomerAudit` table, a `CustomerTag` table, a `CustomerSuspension` table, each with its own repository, its own configuration, its own join logic, and its own update path). The `Customer` class either grows to seven hundred lines, or fragments into several smaller classes that still need to be loaded together, or both. Each new feature touches the DTOs, the view models, the controllers, the validators, the mapper layer, the seed data, and the tests.

The ECS approach acquires four new components — `SoftDeleted`, `Tags`, `Suspended`, `Audit` — and four small systems to process them. The existing entities are untouched. The existing systems are untouched. The new features attach to whichever entities want them. The DTOs that serialise customers know which components they care about and project from them.

At this point, the cost asymmetry begins to show. The model approach has paid a *per-feature integration tax* that the ECS approach has not. By month twelve, with twenty such features added, the asymmetry is substantial. By month eighteen — see [Chapter 15](../02_TheModelQuestion/15_TheWall.md) — the asymmetry has compounded into the wall.

## The decision matrix

Below is the comparison I would have wanted, in compact form, the first time I had to make this choice. It is not exhaustive. It is, I hope, useful.

| Concern | Models | ECS |
|---|---|---|
| Initial setup | Minimal | Modest (framework, registration, conventions) |
| Cost of feature #1 | Very low | Higher than models |
| Cost of feature #50 | High (integration tax) | Low (additive) |
| Cross-cutting concerns | Awkward (base classes / interfaces) | Native (components + systems) |
| Cross-entity queries | Excellent (SQL / LINQ) | Workable (entity-id components) |
| Transactions | Built-in (ORM, DB) | Manual (custom mechanism) |
| Developer familiarity | Very high | Very low |
| Tooling and ecosystem | Mature, vast | Modest, growing |
| Testing | Variable (fixtures, mocks) | Easy (pure systems, plain components) |
| Parallelism | Hard (hidden state) | Easy (stateless systems) |
| Performance at scale | Variable | Strong (cache-friendly, no hidden work) |
| Onboarding new starters | Familiar pattern, quick | Unfamiliar pattern, a week or two |
| Hot reload | Limited | Excellent |
| Schema evolution | Migrations + careful versioning | Additive components |
| Debuggability | Familiar, well-trodden | Different, but data is plain |

A table is a poor place to capture nuance, so let me follow it with the conditions under which each approach earns the asterisk.

## Where the model approach wins, clearly

**The small, stable CRUD application.** Twenty entities. A handful of operations per entity. A schema that changes slowly. A team that knows the model approach in its bones. *Pick the model.* The fixed cost of ECS — the framework, the registration, the conventions, the onboarding — has no payback at this scale. The model is the simpler answer to the simpler problem.

**The application where the database is the centre of gravity.** A reporting tool. An ETL pipeline. A regulatory submission system. Anything where the operations are *"read these rows, transform them slightly, write them back"*, and where the database does the heavy lifting via stored procedures, materialised views, or pre-computed indexes. *Pick the model.* ECS adds an abstraction layer between the application and the data without adding much value, because the value is in the data layer itself.

**The team unfamiliar with ECS and unable to invest in learning it.** A small team. A tight deadline. An existing model-based codebase that needs feature work. *Pick the model.* The familiarity cost of ECS is real and it lands on the people who can least afford it. The first six months of an ECS codebase, with a team learning the pattern, are slower than the equivalent six months of model work.

**The system whose central operation is a complex relational query.** A financial reconciliation engine. A booking system with intricate availability calculations. A planning tool. *Pick the model.* SQL and a relational ORM are extraordinarily good at this kind of work; ECS, while capable of modelling the same relationships, is fighting an architecture that was not designed for join-heavy lookups.

## Where the ECS approach wins, clearly

**The long-lived system with many entities and many cross-cutting concerns.** A platform. A CRM with deep customisability. A multi-tenant SaaS with many feature flags. A workflow engine. *Pick ECS.* The per-feature cost asymmetry that begins at month six compounds, and the ECS architecture pays back every additional concern with the same fixed cost rather than the rising cost the model approach incurs.

**The system that needs runtime extension.** A plugin architecture. A scriptable simulation. A system where customers can configure their own entity types. *Pick ECS.* Adding a new component and a new system at runtime is a much smaller operation than adding a new entity class at compile time.

**The system that needs determinism, replay, or time-travel.** Anything with regulatory replay requirements. Anything with sophisticated debugging needs. Anything where *"what did the world look like at 10:42 last Tuesday?"* is a real question. *Pick ECS.* Flat components and stateless systems are the natural substrate for these properties; reconstructing them on top of an OO codebase is a significant engineering exercise.

**The system that needs parallel execution of independent work.** A simulation. A game. A scientific computation. A high-throughput data pipeline. *Pick ECS.* The stateless-systems property gives you free parallelism that the model approach can only achieve through considerable careful engineering.

**The system that has hit the wall.** A model-based codebase that is currently slow, painful to extend, and politically difficult to refactor. *Consider ECS, carefully.* The right path is rarely a wholesale rewrite — it is usually a strangler-fig replacement of one bounded context at a time. But the structural problems described in [Chapter 15](../02_TheModelQuestion/15_TheWall.md) do not get better with more model engineering; they get better with a different shape.

## The honest middles

Between the clear wins for each side, there is a wide middle ground where the right answer is genuinely *"it depends"*.

A mid-sized B2B SaaS with thirty entities, a small team, and modest extensibility needs is in the middle. A line-of-business application with deep custom-field requirements is in the middle. A system that *might* hit the wall in two years' time but is currently fine is in the middle.

For these, my honest advice is: *consider the team's appetite, the existing codebase's shape, and the next two years' likely workload, and bias slightly towards the approach the team will be happiest learning.* Both approaches will work. The deciding factor is rarely architectural elegance and almost always operational fit.

## The verdict — handed back

This is the chapter where I deliberately decline to deliver a verdict.

You have read Part II. You have read Part III. You know your system, your team, and your future better than I do. The choice between models and ECS is, properly, yours to make.

I will say this much. *If after all of this, you are still unsure whether the cost of learning ECS is worth it for your particular situation, the answer is probably no, this time.* The pattern rewards conviction, and a half-hearted adoption tends to combine the worst of both approaches — model-shaped thinking inside an ECS framework. Better to wait until you can see the wall on the horizon, and then to make a deliberate choice.

> *Anyone who tells you with great confidence which architectural pattern is universally right is, statistically, either selling something, defending something, or has not yet shipped a system that has had to live for five years.*

## What I will not pretend

There is one thing I will not pretend to be neutral about, and it deserves its own paragraph because it has been the quiet thread through every chapter of Part III.

*The cost of extension matters more than people realise, and it gets worse than people estimate.* Whatever else you take from this comparison, take that. The architecture that lets you ship feature N+1 at the same cost as feature N — whichever architecture that turns out to be in your situation — is the one that will let you build something worth building over five years. The architecture that lets each feature ship a little more painfully than the last is the one that will produce, around month eighteen, the feeling that no amount of refactoring quite seems to lift.

Where you land in that comparison is, of course, up to you.

## Recap

- Models and ECS solve different shapes of problem better; neither is universally right.
- Models win clearly for small CRUD, database-centred systems, teams unfamiliar with ECS, and join-heavy workloads.
- ECS wins clearly for long-lived systems with many cross-cutting concerns, runtime extensibility, determinism / replay needs, and parallel execution.
- Between the clear wins is a wide middle where operational fit matters more than architectural ideology.
- The choice is yours. Make it with eyes open.

## Onwards

That closes Part III. From here the series steps sideways — into a topic that turns out, on close inspection, to underlie everything we have discussed about querying, joining, filtering, and shaping data. Part IV is about collections, set theory, and the lovely fact that most of LINQ and most of EF Core are a single mathematical idea wearing different hats. The first chapter of Part IV begins with the unfashionable thesis that *the database is closer to friend than foe*, and proceeds from there.
