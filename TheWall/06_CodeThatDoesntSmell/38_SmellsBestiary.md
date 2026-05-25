# Chapter 38 — The Complete Code Smells Bestiary

This is the consolidated catalogue — the reference chapter the whole series has been quietly assembling. The model smells of [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md), the ECS smells of [Chapter 21](../03_ADifferentWay/21_ECSSmells.md), the branching and exception and nesting smells of this Part, and a further dozen that have not yet had a chapter of their own, all gathered here with symptom, cause, and cure.

A reference chapter is read differently from the rest — dipped into rather than read through — so I have kept the entries terse and the cross-references plentiful. The standing caveat from [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md) applies throughout and bears repeating once: *a smell is a clue, not a conviction.* One smell rarely justifies a refactor; a cluster usually does; and every one of them has a context in which it is, in fact, the right call. Investigate, then decide.

The classic catalogue owes its existence to Martin Fowler's *Refactoring* and to Kent Beck before him; I have organised by the underlying disease rather than alphabetically, because the grouping is where the insight lives.

## Diseases of size

**The god class / god entity.** A class that does too much — hundreds of lines, dozens of methods, a name that has become a category rather than a thing. *Cause:* accretion without subtraction. *Cure:* extract the cohesive clusters into their own types. (Full treatment: [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).)

**The long method.** A method that does not fit on a screen. *Symptom:* you scroll to understand it; it has comment-headed sections (`// validation`, `// processing`, `// notification`). *Cause:* sequential steps that were never given names. *Cure:* each commented section is a method waiting to be extracted and named after the comment.

**The long parameter list.** Five or more parameters. *Cause:* primitive obsession ([Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md)) or a missing aggregate. *Cure:* value objects, or a parameter record.

## Diseases of change

These are the subtle ones — they do not look wrong in a single file, only across time.

**Divergent change.** One class that changes for many unrelated reasons — the `Customer` class that is edited when the tax rules change, *and* when the email format changes, *and* when the loyalty scheme changes. *Symptom:* the same file appears in unrelated pull requests. *Cause:* the class has absorbed multiple responsibilities. *Cure:* split along the axes of change — the things that change for the same reason belong together; the things that change for different reasons belong apart.

**Shotgun surgery.** The mirror image — one logical change that requires editing many files. *Symptom:* *"add a field to the customer"* touches the entity, four DTOs, three view models, two validators, and the seed data. *Cause:* a single concept scattered across the codebase. *Cure:* gather the scattered pieces; this is the by-feature organisation of [Chapter 34](../05_Architecture/34_CleanComposition.md), and — said quietly — it is the pressure that [Chapter 15](../02_TheModelQuestion/15_TheWall.md) identified as the model wall.

**Parallel inheritance hierarchies.** Every time you add a `FooParcel`, you must also add a `FooParcelHandler` and a `FooParcelValidator`. *Cause:* a structure duplicated across three hierarchies that should have been one. *Cure:* collapse the parallel structures, often via composition.

## Diseases of coupling

**Feature envy.** A method more interested in another class's data than its own. *Cure:* move it to the data. ([Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).)

**Inappropriate intimacy.** Two classes that know too much about each other's internals. *Cure:* extract the shared concern. ([Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).)

**The message chain / train wreck.** `a.B.C.D.E` — a walk through the object graph. *Cure:* expose what the caller wants on the nearest owner. ([Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).)

**The middle man.** A class whose every method just delegates to another class, adding nothing. *Symptom:* `public X DoThing() => _other.DoThing();` repeated for every method. *Cause:* an abstraction that has stopped earning its keep, or never did. *Cure:* delete the middle man and talk to the real object — unless the indirection is a genuine seam (a boundary, an adapter), in which case it stays.

**Data clumps.** The same group of parameters travelling together everywhere — `string street, string city, string postcode` appearing in five signatures. *Symptom:* the same three or four values, always together, never apart. *Cause:* a value object that was never extracted. *Cure:* `Address`, once, used everywhere the clump appeared. (A close relative of primitive obsession, [Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md).)

## Diseases of clarity

**Magic numbers and strings.** A literal `0.5m`, `"GBP"`, `42` embedded in logic with no name. *Symptom:* the reader has to infer what `* 0.5m` means. *Cause:* the value was obvious when written and is obscure forever after. *Cure:* a named constant, or — for the domain values — a value object ([Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md)).

**Comments as deodorant.** A comment explaining what a confusing piece of code does, applied like air freshener over the smell instead of cleaning it up. *Symptom:* `// this is complicated because...`. *Cause:* code that was hard to read, papered over rather than clarified. *Cure:* refactor the code until the comment is unnecessary; the best comment is a well-named method.

**Boolean blindness.** A `true`/`false` at a call site whose meaning has been lost. *Cure:* enum, separate methods, or options record. ([Chapter 37](./37_Branching.md).)

**Dead code.** Code that is never executed — a method nobody calls, a branch that cannot be reached, a feature flag that has been permanently off for a year. *Symptom:* the IDE greys it out, or a coverage report shows it never runs. *Cause:* a change that removed the caller but not the callee. *Cure:* delete it. Source control remembers it if you ever want it back; the living codebase should contain only living code.

## Diseases of premature cleverness

**Speculative generality.** Abstraction built for a future that has not arrived and may never — the `IProviderFactoryStrategy` with one implementation, the configuration option nobody sets, the extension point nobody extends. *Symptom:* abstraction with a single concrete user, named for flexibility it does not yet need. *Cause:* designing for imagined requirements. *Cure:* delete the abstraction; reintroduce it the day the second case actually appears. (*"Three similar things"* is the threshold for an abstraction; *"one thing and a guess"* is not.)

**Premature optimisation.** Code contorted for performance that was never measured to be a problem — the hand-rolled loop that replaced a clear LINQ query to save microseconds on a path that runs twice a day. *Symptom:* unclear code justified by an unsubstantiated performance claim. *Cause:* optimising on instinct rather than measurement. *Cure:* revert to the clear version; measure; optimise only what the profiler actually flags. (Note the careful boundary: the *structural* performance the series has advocated throughout — choosing the right collection, projecting before materialising — is not this; this is the *local* contortion of clear code for unmeasured gain.)

**The temporary field.** A field that is only set and meaningful some of the time, null or default the rest. *Symptom:* a field with a comment *"only valid during processing."* *Cause:* a variable that wanted to be a method parameter or a separate type's field. *Cure:* pass it as a parameter, or extract the transient state into its own object (and recall, from [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md), that conditionally-meaningful fields often want a discriminated union).

## Diseases specific to the architectures

For completeness, the two architecture-specific bestiaries, which live in full elsewhere:

**Model smells** — the four-master class, the public setter, the anaemic-no-validation entity, the `Manager` suffix, the bidirectional navigation, and the rest — are catalogued in [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md).

**ECS smells** — the bloated component, the stateful system, the over-componentisation, the implicit system ordering, the orphan entity — are catalogued in [Chapter 21](../03_ADifferentWay/21_ECSSmells.md).

## A field guide to using the field guide

The temptation, with a catalogue like this in hand, is to go through a codebase ticking off smells and filing refactoring tickets for each. Resist it, gently.

Smells are most useful not as a checklist but as *vocabulary* — a shared language that lets a code-review comment say *"this looks like shotgun surgery, can we gather it?"* instead of gesturing vaguely at a feeling. They are most useful *in clusters* — one boolean parameter is fine, but a boolean parameter on a god method with a magic number and a comment-as-deodorant is a method telling you a clear story about how it got that way. And they are most useful *with judgement* — every entry in this bestiary has a context in which it is the right call, and a catalogue applied without judgement produces the kind of mechanically "clean" code that satisfies a linter and pleases no human reader.

> *Refactoring a codebase by mechanically eliminating every catalogued smell, without judgement, produces something rather like a hedge trimmed by someone working strictly from a diagram — geometrically impeccable, eerily uniform, and somehow less alive than the slightly shaggy thing it replaced.*

## Recap

- This is the consolidated reference; the model and ECS bestiaries ([Ch. 14](../02_TheModelQuestion/14_ModelSmells.md), [Ch. 21](../03_ADifferentWay/21_ECSSmells.md)) complete it.
- The diseases group by underlying cause: size, change, coupling, clarity, and premature cleverness.
- The change-diseases (divergent change, shotgun surgery) are the subtle ones — invisible in one file, obvious across time.
- Speculative generality and premature optimisation are the cleverness traps; the cure for both is *wait for the real requirement*.
- Smells are vocabulary and clues, most useful in clusters and with judgement — not a checklist to mechanically satisfy.

## Onwards

That closes Part VI, and very nearly closes the book. What remains is to take everything — the types, the models, the components, the sets, the queries, the architecture, the daily habits — out for a single drive. Part VII is a case study: one problem, built end to end, with the principles of the whole series applied. And then, a short farewell.
