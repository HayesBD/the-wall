# Chapter 34 — The Art of Clean Composition

We have spent this Part on the pieces — dependencies, middleware, services, utilities. This chapter is about the whole: how the pieces fit together into an application that a person can hold in their head, navigate without a map, and change without fear. It is, in a sense, the chapter the whole series has been building towards at the level of *architecture* — the small disciplines that keep a codebase comprehensible long after it has grown too large for any one person to have read all of.

I want to be careful to distinguish this from architecture-with-a-capital-A — the world of formal patterns, layer diagrams, and the kind of document that gets printed and pinned to a wall and then quietly ignored. The composition I mean is smaller and more daily. It is the set of habits that determine whether a new starter, dropped into the codebase, can find the thing they are looking for on the first guess.

## The test of clean composition

Here is the test, and it is a more demanding one than it sounds: *can a developer who has never seen this codebase correctly guess where a given piece of code lives, on the first try, more often than not?*

If the answer is yes, the composition is clean, whatever its formal architecture. If the answer is no — if finding the order-cancellation logic requires a full-text search and three wrong guesses — then no amount of architectural sophistication elsewhere will save the daily experience of working in it.

Clean composition is, in the end, about *predictability*. The reader should be able to predict where things are, what they are called, and what depends on what, without being told. Everything else in this chapter is in service of that one property.

## Organise by feature, not by layer

The most consequential structural decision a codebase makes is whether to organise *by layer* or *by feature*, and the mainstream default — by layer — is, in my view, usually the wrong one.

The by-layer structure groups code by its technical role:

```
Controllers/
    OrdersController.cs
    CustomersController.cs
Services/
    OrderService.cs
    CustomerService.cs
Repositories/
    OrderRepository.cs
    CustomerRepository.cs
Models/
    Order.cs
    Customer.cs
```

To make a single change to how orders work — add a field, change a rule — you touch four folders. The order-related code is scattered across the codebase by technical role, and the four files you need are as far apart as they could possibly be. The folders group things that change *for different reasons* and separate things that change *together*.

The by-feature structure groups code by what it is *about*:

```
Orders/
    OrdersController.cs
    OrderService.cs
    OrderRepository.cs
    Order.cs
    OrderConfiguration.cs
Customers/
    CustomersController.cs
    CustomerService.cs
    CustomerRepository.cs
    Customer.cs
    CustomerConfiguration.cs
```

Now a change to how orders work lives in one folder. The things that change together are together. A new starter looking for *"where does order cancellation live?"* looks in `Orders/` and is, more often than not, right on the first guess. This is the structure that passes the test of clean composition, and it is — not coincidentally — the structure that maps most naturally onto the ECS organisation of [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md), where the components and systems for a concern naturally cluster.

There are exceptions — genuinely cross-cutting infrastructure (logging, the DI composition root, shared middleware) belongs in its own place, because it is *about* the whole application rather than any one feature. But the domain code, the great bulk of it, wants to be organised by what it is about.

## The public surface and the internal scaffolding

A module — a feature folder, a project, an assembly — has two kinds of code: the part other modules are allowed to use, and the part that is nobody else's business. C# gives you the tools to enforce the distinction, and most codebases decline to use them.

The `internal` access modifier means *"visible within this assembly, invisible outside it."* A `public` type is part of the module's contract; an `internal` type is scaffolding. The discipline of marking the scaffolding `internal` — the repository implementation, the helper classes, the intermediate types that exist only to support the module's public operations — means that the *public surface* of a module is small, deliberate, and visible. Another developer can read the public types of a module and know everything they are allowed to depend on, without wading through the implementation.

The codebase that marks everything `public` by default has, in effect, declared that every class is part of every other module's contract — which means no class can be changed without the theoretical risk of breaking a distant consumer, which means change becomes fearful, which means the codebase ossifies. The `internal` keyword, used deliberately, is one of the cheapest available defences against that ossification.

## Dependencies flow one way

A clean composition has dependencies that flow in one direction — the domain at the centre, depending on nothing; the infrastructure at the edges, depending on the domain; the application services in between. The order matters less than the *consistency*: dependencies should flow towards stability, and the most stable thing — the domain model, the business rules — should depend on the least.

The smell that this has gone wrong is the *circular dependency*: module A depends on module B, which depends on module A. The compiler will sometimes permit this (within an assembly) and sometimes forbid it (across assemblies), and the forbidding is a feature — it is the build system enforcing a discipline that keeps the dependency graph navigable. A codebase split into assemblies along feature lines gets circular-dependency detection for free, which is one of the underrated reasons to split a large codebase into more than one project.

## Naming as architecture

The last and least technical discipline, and perhaps the most important: *names are the primary interface to a codebase*, and good names do more architectural work than most diagrams.

A class called `OrderCancellationService` tells the reader what it does. A class called `OrderManager` does not — it tells the reader only that it does *something* with orders, and that the author could not be more specific. The `Manager`/`Helper`/`Util` suffixes we catalogued in [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md) are, in this light, composition smells as much as naming smells: they signal a piece of the system whose responsibility was never pinned down, and a responsibility that was never pinned down is a responsibility that will, over time, accrete everything in its vicinity.

The discipline: *name things for what they do, specifically, and when you cannot, treat that as a signal that the thing does too many things to have a single name.* The class you cannot name precisely is the class that wants splitting. Naming is not decoration applied after the design; naming is a tool *for* the design, and the struggle to find a good name is often the design problem announcing itself.

## Performance, in passing

Composition is mostly a human concern rather than a machine one, so the performance note here is shorter than usual and slightly oblique.

The connection is this: a cleanly composed codebase is one in which the *performance-critical paths are findable*. When the order-processing logic lives in one place, profiling it, understanding it, and optimising it are tractable. When it is scattered across four layers and a dozen files, even *finding* the hot path is a project, and optimising code you cannot find is a non-starter. Clean composition does not make code faster directly; it makes the code *fast enough to be made faster*, which over the life of a system is the more valuable property.

> *A codebase organised by technical layer rather than by feature is rather like a kitchen in which all the handles live in one drawer, all the blades in another, and all the blunt metal objects in a third — beautifully categorised, defensible on paper, and a small daily ordeal for anyone who simply wants to find a knife.*

## The smells

- A change to one feature that touches four or more folders — the composition is by layer, not by feature.
- Everything marked `public` — the module has no distinction between its contract and its scaffolding.
- Circular dependencies between modules — the dependency graph has lost its direction.
- A class named `...Manager`, `...Helper`, or `...Util` — a responsibility that was never pinned down.
- A new starter who cannot guess where things live — the most direct evidence that composition has failed.
- An infrastructure concern (logging, config) scattered through feature code instead of living in its own place.

## Recap

- The test of clean composition: can a newcomer guess where code lives on the first try?
- Organise by feature, not by layer — keep the things that change together, together.
- Use `internal` to distinguish a module's public contract from its scaffolding.
- Let dependencies flow one way, towards stability; circular dependencies are a navigability failure.
- Names do architectural work; the class you cannot name precisely is the class that wants splitting.

## Onwards

That closes Part V. We have travelled from the individual collaborators of an application up to the shape of the whole. Part VI turns to the smallest scale of all — the daily, line-by-line habits that separate code one is proud of from code one keeps quiet about at parties. Early returns, the proper use of exceptions, the quiet menace of the unnecessary branch, and the consolidated bestiary of every smell the series has yet to formally catalogue.
