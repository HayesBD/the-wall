# Chapter 14 — The Model Code Smells Bestiary

This chapter is part reference, part consolidation. The previous four chapters in Part II have, in passing, named most of the recurring failure modes of model-based code; here we gather them in one place, with their symptoms, their causes, and their cures, so that the next time you find one in a code review you have both a name to call it and a thing to recommend instead.

The idea of a *code smell* — popularised by Martin Fowler's *Refactoring* — is that we are not, mostly, looking at *bugs*. We are looking at surface clues that the *shape* of the code is wrong: not broken, not in violation of any rule, but harder to read, harder to extend, and slightly likelier to acquire a real bug six months from now. Smells are weak signals. A single one rarely justifies a refactor; three or four in the same file usually do.

I have grouped these by what they tell you about the model's underlying problem. The groupings are mine; the smells themselves are mostly inherited from the long literature.

## Group one — structural smells

**The god entity.** An entity class over five hundred lines. *Symptom:* the file is hard to navigate, IntelliSense produces an embarrassing scroll, two developers cannot edit it the same week without merge conflicts. *Cause:* years of accreted behaviour piled onto the original concept, with no one ever removing anything. *Cure:* extract methods to domain services, value objects, or smaller entities; force the entity back down to its core invariants. If you cannot, the entity is probably four entities sharing a name (see also [Chapter 10](./10_WhatIsAModel.md)).

**The long parameter list.** A constructor or method with seven or more parameters. *Symptom:* every call site has a comment explaining which value goes in which slot. *Cause:* usually primitive obsession (see [Chapter 12](./12_PrimitiveObsession.md)) — the parameters are all `string`s and `int`s — or a missing aggregate that would have grouped them. *Cure:* introduce value objects for the related parameters, or a parameter object (a record) for the whole set.

**The boolean trap parameter.** A method like `SaveOrder(true)` or `UpdateCustomer(false, true)`. *Symptom:* the call site is unreadable without jumping to the definition. *Cause:* a method that does two related things, distinguished by a flag. *Cure:* split into two methods with names that say what they do, or pass a small enum or `record` of options. *(`Save(SaveOptions.AsDraft)` is fine; `Save(true)` is not.)*

**The mixed-concerns class.** A `User` class with `[Key]`, `[JsonProperty]`, `[Display]`, and a `Validate()` method. *Symptom:* serialisation attributes mixed with persistence attributes mixed with UI hints mixed with domain logic. *Cause:* one class trying to be four (see [Chapter 10](./10_WhatIsAModel.md)). *Cure:* separate into domain, persistence, DTO, and view-model types; translate between them at the boundaries.

## Group two — behavioural smells

**Feature envy.** A method on class `A` that mostly accesses `B`'s data. *Symptom:* half the method body is `that.SomeProperty` or `that.GetSomething()`. *Cause:* the method is in the wrong place. *Cure:* move the method to `B`, where the data lives.

**The train wreck.** A line that walks a chain of navigation properties: `customer.LatestOrder.Items.First().Product.Manufacturer.Name`. *Symptom:* a series of dots, each requiring the previous one to be non-null. *Cause:* the caller has too much knowledge of the internal shape of the graph; also, every step is a chance for a `NullReferenceException` or, if you are using EF Core with lazy loading, an unexpected database query. *Cure:* expose what the caller actually wants as a single method or property on the closest natural owner — `customer.LatestOrderManufacturerName`, perhaps, or a projection.

**Inappropriate intimacy.** Two classes that each access the other's private members through `internal` visibility or convenient public properties that were *"really meant to be private"*. *Symptom:* changing one almost always requires changing the other. *Cause:* a missing third concept that should have owned the shared behaviour. *Cure:* extract the shared concern into its own type that both depend on.

**Refused bequest.** A subclass that overrides most of its parent's methods to throw `NotImplementedException` or to do nothing. *Symptom:* `throw new NotSupportedException("Not applicable for this type")` appearing in five overrides. *Cause:* the inheritance relationship is wrong — the subclass is not really a kind-of the parent. *Cure:* replace inheritance with composition, or replace the hierarchy with a discriminated union (see [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)).

## Group three — encapsulation smells

**The public setter on an entity.** `public string Email { get; set; }`. *Symptom:* any piece of code anywhere can mutate the entity to any state, valid or invalid. *Cause:* defaulting to `{ get; set; }` because it is what scaffolding produces. *Cure:* `{ get; private set; }` or `{ get; init; }`, with a method on the entity that performs the change and enforces the rule (`customer.ChangeEmail(newEmail)`).

**The anaemic model with no validation.** A `record` with thirty properties and no constructor logic, no validation, no consistency checks. *Symptom:* you can construct an instance with values that the domain considers nonsense. *Cause:* the rules live elsewhere — usually in handlers, sometimes nowhere. *Cure:* depending on which side of the rich-vs-anaemic debate (see [Chapter 11](./11_RichVsAnemic.md)) you sit, push the validation into the entity's constructor, or formalise it in a `Validator<T>` that runs reliably at every entry point.

**The conditionally meaningful nullable.** A class with `int? CompletedQuantity`, `DateTime? CompletedAt`, `string? CompletionNotes`, all of which are only set when `Status == Completed`. *Symptom:* comments saying *"only set when..."*. *Cause:* a discriminated union (see [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)) pretending to be a class with nullable fields. *Cure:* separate types for each state; pattern-match at the consumption point.

**The entity that knows about the framework.** A domain method that takes an `HttpContext`, an `ILogger`, or an `IConfiguration`. *Symptom:* the domain is reaching upwards into the infrastructure layer. *Cause:* convenience under deadline. *Cure:* pass plain values; if you genuinely need a service, the operation belongs in a handler, not on the entity.

## Group four — naming smells

**The `Manager` / `Helper` / `Util` suffix.** A class called `UserManager`, `OrderHelper`, or `DateUtil`. *Symptom:* the suffix tells you nothing about what the class does, only that the author could not think of a better name. *Cause:* the class lacks a clear single responsibility, often because it has accreted unrelated methods over time. *Cure:* name what the class actually does. If you cannot, split it. *(If you find a `Manager` class with twenty methods, you have probably found ten classes wearing one name.)*

**The `Data` / `Info` / `Details` suffix.** `OrderData`, `CustomerInfo`, `ProductDetails`. *Symptom:* the suffix is doing the job that the name should be doing. *Cause:* a DTO or projection that the author did not want to call `OrderResponse` or `OrderSummary`. *Cure:* name it for what it represents in context.

**The redundant prefix.** `Customer.CustomerName`, `Order.OrderTotal`. *Symptom:* every property repeats the class name. *Cause:* a hangover from older codebases or generated SQL columns. *Cure:* `Customer.Name`, `Order.Total`. The context is the class.

## Group five — persistence-related smells

**The bidirectional navigation kept for symmetry.** `Customer` has `Orders`; `Order` has `Customer`. Both navigation properties exist; only one is actually used. *Symptom:* lazy loading firing queries you did not expect; serialisation cycles; EF tracking overhead. *Cause:* a reflex to make graphs "complete" when the application's read pattern is one-directional. *Cure:* keep only the direction you actually use. The foreign key on the `Order` row is what makes the join work; the navigation property is only sugar.

**The entity that lazy-loads its way into a transaction problem.** A method body that touches several navigation properties in sequence, each one firing its own query, none of them under a single explicit `Include`. *Symptom:* a single business operation producing twelve database round-trips. *Cause:* lazy loading enabled by default and trusted blindly. *Cure:* explicit `Include`s for what you need; projection (`Select(...)`) when only a subset is required. See [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md) for the full treatment.

**The entity reaching for a repository.** A method on `Customer` that takes an `IOrderRepository` parameter. *Symptom:* a domain method with infrastructure dependencies. *Cause:* the operation is really an orchestration, not an entity behaviour. *Cure:* move the operation to a service or handler; let it call the repository and pass the loaded entities to a smaller, repository-free method on `Customer`.

**The N+1 inside the constructor.** A constructor or factory method that fires a query per object during materialisation. *Symptom:* loading a list of ten things takes eleven queries. *Cause:* a `foreach` over `entities` that calls a constructor that, in turn, loads related data. *Cure:* load related data in one query; hand it in.

## A small word on smell discipline

Smells are clues, not commandments. A 700-line entity in a stable, well-tested module is sometimes the right thing — better one large class everyone knows than seven small ones nobody can find. A `boolean` parameter is acceptable on a method called `SetEnabled(bool)`. A long parameter list is fine if the parameters are unrelated and packaging them into a record would obscure rather than clarify.

Treat the smell as the opening of an investigation, not the closing of one. *"This has the shape of feature envy — is the data really better off on the other class?"* is a useful question. *"Feature envy detected; refactor mandatory"* is not.

> *Applying refactoring rules without judgement is rather like applying grammar rules without judgement: technically correct sentences that no human being would ever, of their own free will, choose to read.*

## Performance, in passing

A respectable share of these smells have measurable performance costs as well as readability ones. The N+1 query is the headline example — twelve database round-trips where one would have done — but the train wreck through lazy-loaded navigation, the bidirectional navigation kept for symmetry, the god entity that drags forty columns to memory when six would have answered the question, and the constructor that fires queries during materialisation are all, in addition to being aesthetically displeasing, *slow*.

Cleaning up the smells almost always makes the code faster as well as clearer. It is one of the more reliable cases of the performance thread paying off without anyone setting out to make it.

## Recap

- A code smell is a surface clue that the shape of the code is wrong, not a guarantee that anything is broken.
- The recurring model smells fall into five groupings: structural, behavioural, encapsulation, naming, and persistence-related.
- Most of them have a cleaner alternative covered elsewhere in this Part or the previous one; cross-reference, do not memorise.
- Smells are clues, not commandments — investigate, then decide.
- Many of the smells have a performance cost in addition to the readability cost; cleaning them up usually pays back twice.

## Onwards

The next chapter is the most honest in Part II. We have, so far, treated models with care, presented their best case, given each smell its alternative, and assumed a disciplined team applying everything we have said. [Chapter 15](./15_TheWall.md) asks the question every working developer eventually has to answer: *what happens to even the most carefully built model-based codebase eighteen months in?* The answer is not, I think, the one most of us were trained to give.
