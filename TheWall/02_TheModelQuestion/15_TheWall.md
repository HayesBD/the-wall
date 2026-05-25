# Chapter 15 — The Wall: When Models Stop Scaling

I want to make a confession before we begin.

The previous five chapters of Part II have been, in their way, optimistic. We laid out a definition of *model*. We treated the rich-vs-anaemic debate fairly. We retired primitive obsession with value objects. We introduced the two-identifier pattern. We catalogued the smells and their cures.

If you applied every piece of advice in those chapters, with the discipline of a senior engineer who reads code reviews at half past ten on a Friday evening and never lets a bad pattern slip through, you would have a codebase that worked beautifully for somewhere between twelve and eighteen months.

And then, around month eighteen — give or take, depending on your team, your domain, and the rate at which your product manager is allowed to dream — something quiet and disagreeable would begin to happen. The metrics would not, at first, be alarming. Velocity would still be respectable. New features would still ship. The dashboards would still be green. But the *feeling* in the team would have changed, in a way that a thoughtful tech lead could not quite put into words and a stakeholder could not see at all.

This chapter is about that feeling, and about what causes it.

## The shape of the slowdown

Let me describe the symptom set, gathered from a fairly wide sample of teams.

A new feature is requested. It sounds small. Add a `Tier` to a customer; promote customers automatically when their lifetime spend crosses a threshold; let support staff override the tier manually with an audit trail.

The senior engineer estimates two days. The senior engineer is correct to the hour about the *coding*. The senior engineer is wildly wrong about the *integration*.

The actual work, by the time it ships ten days later, looks something like this:

- Add a `Tier` enum to `Customer`. *(Half a day.)*
- Update the `Customer` entity's constructor and the `IEntityTypeConfiguration`. *(An hour.)*
- Update the seven DTOs that already serialise `Customer`. *(A day, including three rounds of code review.)*
- Update the four view models that already render `Customer`. *(A day.)*
- Add the lifetime-spend calculation. Discover it requires joining `Orders` which requires updating the `OrdersController` projection, which is shared with the reports module. *(Two days.)*
- Add the manual override. Discover the audit trail is on a separate `AuditEntry` table that requires a new entity, a new repository, a new migration, a new write path, and a small disagreement at standup about whether audit entries should be in the same `DbContext`. *(A day and a half.)*
- Add the automatic-promotion rule. Discover three places already mutate `Customer` and the rule needs to fire from all three. *(A day.)*
- Realise that the existing `CustomerService` is now nine hundred lines and split it. *(Half a day.)*
- Realise that the split has broken six tests in subtly different ways. *(Half a day.)*
- Realise that one of the broken tests was actually broken before but was passing for the wrong reason. *(Half a day, ending in a small moment of existential dread.)*

This is not a story about an incompetent team. This is a story about *the model becoming load-bearing for things it was never designed to support*. Every one of those steps was reasonable in isolation. The cumulative effect is the wall.

## Why this happens, and why the smells chapter was insufficient

The previous chapter catalogued the smells. The implicit promise of a smells chapter is that, if you avoid the smells, you avoid the trouble. The honest disclosure I have to make in this chapter is that *the smells chapter, on its own, does not save you*. You can have a codebase with zero god entities, zero boolean traps, zero anaemic-with-no-validation classes, every smell scrupulously refactored, and still hit the wall.

What does the wall actually consist of?

**Conceptual coupling that the type system cannot see.** `Customer`, `Order`, `Product`, and `Tier` are separate classes. They are not, conceptually, separable. A change to `Customer.Tier` requires understanding what `OrdersController` projects, what the `LoyaltyEmailJob` reads, what the `ReportingService` aggregates, and what the front-end's `CustomerCard` component renders. None of these dependencies appear in any class declaration. They emerge from years of small decisions, each one locally sensible, none of them tracked anywhere a compiler can read.

**The cross-cutting concern that is not, in fact, cross-cutting.** Logging, authorisation, and validation are usually called cross-cutting concerns and are usually solved with middleware, attributes, or interceptors. But there is a second category of cross-cutting concern that mainstream OO has never had a good answer for: *concerns that apply to a subset of entities chosen by some runtime rule*. Soft delete. Tenancy. Audit. Optimistic concurrency. Versioning. Tagging. Each one wants to apply to *some* entities and not others, and the decision of which is which is rarely stable. The mainstream answer is *base classes*, which collapses into the god-base-class problem by month nine.

**The extension that requires changing six files.** Adding a new *kind of customer* — a corporate customer, say, with different tax rules and a different invoicing flow — touches the entity, the configuration, the DTOs, the view models, the controllers, the validators, the mappers, the tests, and the seed data. Nine files for a concept the business person described in a single sentence. Adding the *eleventh* such kind, a year later, is not nine times worse — it is more than nine times worse, because the cross-coupling among the previous ten is now part of the system.

**The hidden state in the change tracker.** EF Core's change tracker is brilliant for ninety percent of cases and a quiet horror for the other ten. An entity loaded for one reason and modified inadvertently elsewhere gets saved when the unit-of-work commits. An entity you wanted to discard ends up persisted because someone, somewhere, called `SaveChanges`. The mental model required to reason about *which entities are tracked, when, and by what* grows with the codebase, and at no point does a colour-coded warning appear on the dashboard to tell you it has grown too large.

**The behaviour-placement debate, ten years on.** The rich-vs-anaemic question (see [Chapter 11](./11_RichVsAnemic.md)) is a fair-minded debate when the codebase is young. By year three, the answer is usually *"we decided to be rich, and we have a four-hundred-line `Order` class"* or *"we decided to be anaemic, and we have eighty handlers each touching `Order` slightly differently"*. The original choice has become its own failure mode. Refactoring out is expensive, slow, and politically fraught.

## We have known about the disease for fifty years

I should pause here and be honest about what is, and is not, new in this chapter — because a reader who has been around will already be muttering the names.

That software *rots* — that a codebase grows more complex and more expensive to change the longer it lives, unless someone spends real effort pushing back — is one of the oldest and best-established observations in the field. Meir Lehman formalised it across the 1970s and 1980s as his *Laws of Software Evolution*: a system in use must continually change, and as it changes its complexity increases unless work is done to reduce it. The same phenomenon wears other names in other decades — *software entropy*, *code rot*, Ward Cunningham's *technical debt*, and Foote and Yoder's gloriously-named *Big Ball of Mud*. None of this is my discovery, and I would be embarrassed to pretend otherwise. If the only claim of this chapter were *"software gets messy over time,"* it would not have earned the page.

The claim is narrower, and I think more useful. It is a *diagnosis*, not an observation. The famous names describe the disease — the rising cost of change — and prescribe a famous treatment: discipline, refactoring, paying down the debt. This chapter argues something more specific, and more uncomfortable: that for a particular and very common class of system, *the disease has a structural cause that no amount of the famous treatment cures*. The debt metaphor says *"you have been sloppy; tidy up."* The structural claim, which the rest of this chapter sets out, says *"you have been tidy, and you are still going to hit the wall, because the shape was wrong for the load."* Those are different claims — and the second is the one this book exists to make, and, in [Chapter 39](../07_TogetherNow/39_CaseStudy.md), to measure.

## What this is not

I do not want to overstate the case.

It is not that model-based architectures *don't work*. They do, and they work for an enormous fraction of the systems running the world today. The CRUD application with twenty entities, a small team, and a reasonably stable domain is well-served by a model approach forever. The internal admin tool, the regulatory reporting system, the line-of-business form-over-database — these are model problems, and a model solution is the right one.

It is also not the case that good engineering does not help. Discipline matters. Bounded contexts (see [Chapter 10](./10_WhatIsAModel.md)) matter. Value objects matter. Lean DTOs matter. The two-identifier pattern (see [Chapter 13](./13_OnIdentifiers.md)) matters. A team that applies these consistently will hit the wall later, less hard, and with more options when they do.

What I want to land is this: *for a particular class of system, the wall is structural*. It is not a discipline problem. It is not a refactoring problem. It is a problem of *the model being asked to be the unit of extension when the natural unit of extension is something smaller and more composable*. Once you see it from that angle, the discipline-based fixes — extract this class, rename that one, push these methods down — begin to feel like rearranging the deck chairs on something that was, structurally, never going to scale gracefully past a certain size.

> *Asking a model-based architecture to scale to a hundred entities with composable cross-cutting behaviour is rather like asking a Victorian terrace to scale to a thousand residents: you can keep adding extensions, you can keep splitting rooms, you can re-plumb the basement on a Saturday afternoon — but at some point the honest answer is that the shape itself was never intended for what you are now asking of it.*

## The shape that succeeds at scale

I will not name the alternative in this chapter. That is the work of Part III. But I will, since we have spent the chapter being honest, sketch the shape of the answer.

It is an answer that treats *behaviour* as something that can attach to a runtime selection of entities rather than something baked into a class declaration. It is an answer that lets you add a *new kind of thing* — a new cross-cutting concern, a new entity kind, a new feature — by writing code that affects only the parts of the system that the new thing actually touches, with no edits to the entities themselves. It is an answer in which the *cost of adding feature N+1* does not grow as N grows, because feature N+1 lives in its own place and is composed in at runtime rather than woven through the entity hierarchy at compile time.

This is not the answer for every system. It is, I think, the answer for the kinds of system that hit the wall described above. We will spend Part III examining it carefully, fairly, with the same scepticism we applied to models in Part II, and at the end of it — in [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md) — we will put the two side by side and let the reader judge.

## Performance, in passing

A note that may or may not surprise. Codebases that hit the wall described above are *almost always* slow as well as hard to extend. The reasons are structural rather than incidental: god entities materialise more data than the use-case needs; the change tracker tracks more entities than the use-case requires; the projection that has to satisfy seven DTOs is heavier than any one of them needs; the lazy-loaded navigation property that should have been an explicit `Include` fires a query the use-case never asked for.

The performance cost is, in a sense, the *visible shadow* of the structural problem. The structural problem is the cause; the slow page load is the symptom. Teams that fix the structural problem usually find, as a by-product, that several of their longest-standing performance issues simply evaporate. The dashboard goes green without anyone setting out to make it.

## The smells (of an entire codebase, not a single file)

- Estimates that are routinely off by 3-5x for "small" features, with the slippage almost entirely in integration rather than coding.
- A documented "places to remember to update" list for the most-touched entities.
- A standup conversation in which adding a new entity kind is described as *"a project"*.
- A six-month freeze on refactoring because *"we don't have the runway to break things"*.
- New starters who are productive on isolated bug fixes within a week but cannot ship a cross-cutting feature for two months.
- A senior engineer who can describe the entity graph from memory and is, by quiet consensus, indispensable.

## Recap

- Even a well-disciplined model-based codebase tends to hit a structural wall around month eighteen.
- The wall is not caused by the smells catalogued in [Chapter 14](./14_ModelSmells.md); it persists despite their absence.
- The underlying cause is the model becoming the unit of extension when the natural unit of extension is smaller and more composable.
- Model-based architectures remain the right answer for many systems — particularly the small, stable, CRUD-shaped one. They become the wrong answer for the large, evolving, behaviour-rich one.
- The performance problems that accompany the wall are usually the structural problem casting a shadow into the dashboard.

## Onwards

Part III begins. We turn to an architecture that grew up in a quite different corner of the field — high-performance game engines, where the wall described above was hit twenty years earlier and a quite different answer had to be found. It is an answer that, applied with care to the kinds of system that hit the wall, deserves a great deal more attention from mainstream business software than it has so far received. Let us begin.
