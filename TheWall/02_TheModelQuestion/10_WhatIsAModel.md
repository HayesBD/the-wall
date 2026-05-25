# Chapter 10 — What Is a Model, Really?

We need to start by clearing some semantic underbrush.

The word *model*, in the world of object-oriented software, has been worked so hard that it has more or less lost the ability to mean any one specific thing. A request for *"the user model"* in a code review can produce, depending on which colleague you have asked, an EF Core entity class, a JSON-serialisable DTO, a view model bound to a form, a domain object with rich behaviour, or an anaemic record with five properties and a serialisation attribute on each. All five of these are commonly called *the model*. All five are, in subtle but important ways, different things.

This chapter is an attempt to lay out what *I* mean, in this series, when I use the word — and, more importantly, what I think any working developer should mean by it most of the time. I will not insist you adopt my terminology. I will insist that you adopt *some* terminology, because the alternative is the meeting where four people use the same word to mean four different things and the loudest one wins.

## The five things people mean

Let us begin by naming the five.

**The domain model.** A class (or set of classes) that represents a thing in your business domain — a customer, an order, a booking, an invoice — together with the behaviour, the invariants, and the rules that apply to it. This is the model in the original object-oriented sense: a small simulation of the world your software lives in.

**The persistence model.** A class shaped to fit a row in a database table — usually with an `Id`, foreign keys, navigation properties, perhaps some annotations or fluent configuration. Its job is to round-trip cleanly between memory and disk. It exists for the database.

**The DTO — data transfer object.** A class designed for a wire format — a JSON request body, a JSON response body, a message on a queue. Its shape is dictated by what some other system needs to send or receive. It exists for the wire.

**The view model.** A class shaped to fit a particular screen, form, or component. Its properties match what the UI needs to display or collect. It exists for the user interface.

**The anaemic model.** A class whose only purpose is to hold properties — no behaviour, no methods of substance, just data. Whether this counts as a *model* at all is one of the longer-running debates in the field, and we will give it the attention it deserves in [Chapter 11](./11_RichVsAnemic.md).

These are five distinct concepts. They serve five distinct purposes. They are, in the wild, almost always implemented as the same class.

## Why one class for all five is the problem

Picture, if you will, the `User` class in a moderately mature codebase. It started life as a domain model — a `User` had a name, an email, and a method to change their password. It then became a persistence model, because the team adopted Entity Framework and the easiest thing was to put `[Key]` on the `Id`. It then became a DTO, because the API needed to return user data and the easiest thing was to serialise it directly. It then acquired several view-model concerns, because the user-edit form needed a `ConfirmPassword` field and a `RoleNames` list and the easiest thing was to put those on the user as well.

The result, after eighteen months, is a class with thirty-odd properties, half of which are nullable and conditionally meaningful, an `[NotMapped]` on five of them so EF Core does not get confused, a `[JsonIgnore]` on six of them so the API does not return the password hash, and a constructor whose preconditions nobody could now explain under oath. Velocity on the user-related parts of the codebase has halved without anyone noticing exactly when.

> *Asking one class to play four roles is the architectural equivalent of asking the same person to be the chef, the waiter, the accountant, and the health inspector — they will, eventually, fail at all four in interestingly correlated ways.*

This is the model problem in microcosm. It is not that any one of those changes was wrong on its own. It is that *the same class was asked to serve four masters* — the domain, the database, the wire, and the UI — and four masters with conflicting requirements will, over time, contort the class into a shape that serves none of them well.

## The remedy, in outline

The remedy is to separate the five concerns into separate types and to translate between them at the boundaries.

```
HTTP request ──► DTO ──► domain model ──► persistence model ──► DB
                                  │
                                  └─► view model ──► screen
```

Each type is small. Each type has one job. The translation between them is explicit, and lives in a small number of well-defined places (request validators, mappers, repositories, view-model factories).

If this sounds like more work, it is. The pay-off is that none of the four masters can subsequently push their requirements onto the others. The DTO can change to match a new API contract without disturbing the domain. The persistence model can be reshaped for a schema migration without disturbing the wire. The view model can adopt a new front-end framework's preferred shape without anyone touching the domain at all.

It is also worth noting, briefly and with the discipline of the recurring performance thread, that this separation tends to be *faster* than the all-in-one approach. A DTO that contains only the eight fields the API actually needs serialises and deserialises faster than a thirty-property kitchen-sink class. A persistence model that excludes view-only computed fields makes for cheaper queries. The performance wins are not the reason to separate, but they are a quiet reward for having done so.

## The bounded context, in one paragraph

The discipline of separating models has a senior name in the field — Domain-Driven Design's *bounded context*. The idea is that a single concept like *user* may legitimately be modelled differently in different parts of a large system: a `User` in the authentication subdomain is mostly an identity and a credential; a `User` in the billing subdomain is mostly an account and a payment method; a `User` in the support subdomain is mostly a ticket history. Trying to force these into a single class produces precisely the bloat described above. Treating them as separate models that happen to share an identifier produces a system that scales.

We will not pursue DDD any further in this series — it is a large topic and there are better books for it — but the underlying instinct, *let the model fit the context*, is the same instinct that drives everything in Part II.

## Performance, in passing

A small note that, again, will recur. Lean, purpose-built models are faster than fat, all-purpose ones — serialisation is cheaper, validation is cheaper, in-memory copies are cheaper, EF Core's change tracking is cheaper. The biggest win is often EF Core: a query that materialises ten properties is dramatically cheaper than one that materialises forty, and a `SELECT` that returns ten columns is dramatically cheaper than one that returns forty, all the way back through your database's network buffers. The chapter on EF Core ([Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md)) treats this properly.

## The smells

- A single class with `[Key]`, `[JsonProperty]`, `[Display]`, and a `Validate()` method — you have a four-master class.
- A `User` with a `ConfirmPassword` property because the form needs one — the form's needs have leaked into the domain.
- A property on a domain model that is `[NotMapped]` *and* `[JsonIgnore]` *and* `[Display(AutoGenerateField = false)]` — that property is loudly telling you it does not belong on this class.
- A domain object that takes an `HttpContext` to do its work — the domain has reached upward into the framework.
- A `User.cs` file that is over six hundred lines long — almost certainly four classes welded together.

## Recap

- *Model* is an overloaded word. Five common meanings: domain, persistence, DTO, view, and (controversially) anaemic.
- Trying to serve all of these with a single class is the most common cause of model bloat in a maturing codebase.
- The remedy is separation and explicit translation at the boundaries.
- Lean, purpose-built models are quietly faster as well as easier to maintain.

## Onwards

The next chapter takes the longest-running argument in the field — *rich models vs anaemic models* — and gives both sides a properly fair hearing. By the end of it we will be in a position to talk seriously about behaviour, where it belongs, and why this is a question that becomes harder, not easier, as the system grows.
