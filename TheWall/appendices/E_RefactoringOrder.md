# Appendix E — The Strangler's Path: Cleaning Up a Codebase in the Right Order

Every technique in this book is easy to apply to code you are writing fresh. The harder and more common situation is the one most working developers actually face: *an existing codebase, large, valuable, in production, and not good*. You have read the preceding chapters; you can see a dozen things wrong; and the question that stops most people is not *what* to fix but *in what order, without breaking the thing that is currently paying everyone's salary*.

This appendix is an answer to that question. It is opinionated, it is a default rather than a law, and it is built around one principle that overrides all the specific advice: **never rewrite; always strangle.**

## Why not a rewrite

The instinct, faced with a bad codebase, is the big rewrite — set aside six months, build the new clean version alongside the old, switch over when it is ready. This instinct has a near-perfect track record of disaster, for reasons worth stating plainly.

The old system, however ugly, encodes *years of accumulated bug fixes, edge cases, and hard-won knowledge* — most of it undocumented, much of it invisible until the rewrite fails to reproduce it. The rewrite, meanwhile, is a project with no shippable value until the very end, which means months of cost with nothing to show, which means the business pressure to *"just ship something"* mounts until the rewrite is either abandoned half-finished or rushed out missing the very edge cases that justified it. And all the while, the old system must keep being maintained — so the team is now maintaining two systems and shipping value from neither.

The alternative is the **strangler fig**, named by Martin Fowler after the vine that grows around a host tree, gradually envelops it, and eventually stands on its own where the tree used to be. You wrap the old system, route *new* work through clean implementations, migrate *existing* functionality piece by piece, and ship value continuously throughout. At no point is there a six-month gap with nothing to show. At no point are you maintaining two whole systems. The old code is strangled gradually, one bounded piece at a time, and the day it is finally deleted is an anticlimax rather than a gamble.

Everything that follows assumes the strangler approach. The order below is the order in which to apply this book's techniques *within* that gradual migration.

## The ordering principle

Three rules determine the order, and they compose into the sequence:

1. **Safety net before surgery.** You cannot refactor safely without a way to know you have not broken anything. Tests come first — specifically, *characterisation tests* that pin down what the system currently does (bugs included), so you can tell whether a change altered behaviour.
2. **Observability before structural change.** Before you move the big pieces, you need to be able to *see* what is happening — what is failing, what is slow, what is being called. Fix the things that blind you (swallowed exceptions, missing logs) early, so the later, riskier changes are done with the lights on.
3. **Leaf before root, local before global, reversible before irreversible.** Do the contained, low-risk, easily-undone changes first. They build momentum, they make the code more legible, and they often make the bigger changes visibly easier. Save the structural, touches-everything changes for when the pieces they move are already clean.

From those three rules, the sequence falls out.

## The sequence

**Step 0 — Stop the bleeding.** Before improving anything, stop it getting worse. Agree, as a team, that *new* code follows the book's practices even while the old code does not. The strangler vine has to start growing somewhere; it starts with the next pull request. Without this, you are cleaning a beach during an oil spill.

**Step 1 — Build the safety net.** Characterisation tests at the boundaries — the API endpoints, the key workflows — that capture current behaviour. These are not unit tests of internals (the internals are about to change); they are tests of *observable behaviour* that will survive the refactoring and tell you if it broke anything. This is the single most important step and the one most often skipped. Skipping it is how refactoring acquires its (undeserved) reputation for breaking things.

**Step 2 — Restore observability.** Hunt down and fix the things that hide failure: the empty catch blocks and `catch (Exception)`-and-swallow of [Chapter 36](../06_CodeThatDoesntSmell/36_TryCatch.md), the missing logging, the places where errors vanish. You are not yet fixing the logic; you are making sure that when the *later* steps break something, you find out immediately rather than three weeks later from a customer. Turn on the lights before rearranging the furniture.

**Step 3 — The free wins.** Apply the local, reversible, high-clarity changes that need no structural permission: guard clauses and the flattening of nested pyramids ([Chapter 35](../06_CodeThatDoesntSmell/35_EarlyReturns.md)), the deletion of dead code ([Chapter 38](../06_CodeThatDoesntSmell/38_SmellsBestiary.md)), the renaming of `Manager`/`Helper`/`Util` classes to say what they do ([Chapter 34](../05_Architecture/34_CleanComposition.md)). None of these changes behaviour; all of them make the code more legible; and a more legible codebase makes every subsequent step easier to reason about. This step also builds team momentum — visible, low-risk improvement that makes the next, scarier steps feel achievable.

**Step 4 — Fix the dependency and lifetime bugs.** Now the DI problems of [Chapter 31](../05_Architecture/31_DependencyInjection.md): the captive dependencies (scoped inside singleton), the lifetime mismatches, the over-stuffed constructors. These cause real, intermittent, hard-to-trace bugs, and fixing them is *contained* — a registration change here, a constructor split there — and removes a whole class of heisenbug before you start moving larger pieces. Doing this before the model and query work means the later work happens on a foundation whose object lifetimes you can trust.

**Step 5 — Tame the exceptions and introduce value objects.** With observability restored (Step 2) you can now safely replace exceptions-as-flow-control with result-shaped returns ([Chapter 36](../06_CodeThatDoesntSmell/36_TryCatch.md)), and begin retiring primitive obsession one value at a time ([Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md)). Both are incremental — a value object can be introduced for one concept, with the compiler showing you every site that needs updating, and the change is mechanical and safe. These build the vocabulary the later structural work will use.

**Step 6 — Fix the query layer.** The N+1 problems, the missing projections, the absent `AsNoTracking`, the materialise-then-filter mistakes of [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md). This step has a high and *visible* payoff — pages get faster, the database load drops, the operations team notices — which is valuable for sustaining organisational support for the longer cleanup. It is also relatively contained: query fixes are usually local to a repository or a handler, with the characterisation tests confirming the results are unchanged.

**Step 7 — Separate the models.** Now the bigger, riskier work: pulling the four-master classes apart into domain, persistence, DTO, and view models ([Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md)), moving persistence configuration into `IEntityTypeConfiguration<T>` classes ([Chapter 29](../04_DataAndSets/29_DbContextConfiguration.md)), introducing the `Id`/`PublicId` split ([Chapter 13](../02_TheModelQuestion/13_OnIdentifiers.md)). This touches more and risks more, which is exactly why it comes after the safety net, the observability, and the lifetime fixes are all in place. Do it one bounded context at a time — strangle the `Orders` model while leaving `Customers` alone, ship, then move on.

**Step 8 — Reorganise by feature.** With the pieces clean, restructure from by-layer to by-feature ([Chapter 34](../05_Architecture/34_CleanComposition.md)). This is mechanically large (files move) but logically safe (behaviour does not change), and it is far easier once the models are already separated, because you now know which pieces belong together.

**Step 9 — Consider ECS, where and only where the wall was hit.** Last, and only for the parts of the system that genuinely hit the wall of [Chapter 15](../02_TheModelQuestion/15_TheWall.md) — the areas with many entities, heavy cross-cutting concerns, and a feature-add cost that keeps rising. Here, and only here, the strangler does its most dramatic work: stand up an ECS-shaped implementation of *one* bounded context alongside the model-shaped one, route new functionality through it, migrate the existing functionality piece by piece, and delete the old model when nothing routes to it any more. The reckoning of [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md) governs this decision: most of the codebase will not need it, and the parts that do will have made themselves obvious by being the parts that hurt.

## The shape of the whole

Read down that sequence and notice its arc. It begins with things that change *nothing* about behaviour (tests, observability, renaming) and ends with things that change the *architecture* (model separation, ECS). It begins with the *local* (a guard clause in one method) and ends with the *global* (the shape of the whole system). It begins with the *certain* (a captive dependency is a bug) and ends with the *judged* (ECS is a trade-off). At every step, the system is shippable, the safety net is intact, and the change just made has left the codebase better than it found it.

That arc is the whole method. You are not rewriting the system; you are growing a better one around it, in an order that keeps you safe at every step and shipping value throughout.

> *Rewriting a large legacy system from scratch is rather like demolishing an occupied apartment building in order to construct a better one on the same plot, on the same timeline, without rehousing anybody — ambitious, occasionally attempted, and the source of a great many stories that begin "well, the theory was sound."*

## The order, in one glance

1. **Stop the bleeding** — new code follows the rules.
2. **Safety net** — characterisation tests at the boundaries.
3. **Observability** — fix swallowed exceptions, restore logging.
4. **Free wins** — guard clauses, dead-code deletion, renaming.
5. **Dependencies and lifetimes** — fix captive dependencies and constructor bloat.
6. **Exceptions and value objects** — result types, retire primitive obsession.
7. **Query layer** — N+1, projection, tracking. (Visible payoff.)
8. **Separate the models** — domain / persistence / DTO / view, one context at a time.
9. **Reorganise by feature** — by-layer to by-feature.
10. **ECS, selectively** — strangle only the contexts that hit the wall.

Tests first. Lights on before furniture moved. Local and certain before global and judged. Ship at every step. Never rewrite; always strangle.
