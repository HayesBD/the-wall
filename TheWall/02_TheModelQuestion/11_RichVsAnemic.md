# Chapter 11 — Rich vs Anemic: The Eternal Debate

This is the longest-running argument in the field, and one that has, over two decades of conferences and Twitter spats and impassioned blog posts, generated rather more heat than light. The headline positions are easy to state. The headline positions are also, on close inspection, mostly straw men.

So let me, before saying anything else, lay both cases out at their best.

## The case for the rich model

A *rich* domain model is one where the data and the behaviour belonging to a concept live on the same class. A `BankAccount` has a balance, and it also has `Deposit`, `Withdraw`, `Transfer`, and `Close` methods. The balance is private. The methods enforce the rules — no withdrawal beyond the overdraft limit, no transfer to a closed account, no negative deposit. The state of a `BankAccount` is, at every moment, a state that the class's own code has had the opportunity to validate.

The argument runs:

- **Invariants are protected at the boundary.** No piece of code outside the class can put a `BankAccount` into a state the class did not approve of. The rules live with the data they govern, and they get enforced every time the data moves.
- **The code is discoverable.** A developer who needs to know what can be done to a `BankAccount` types `account.` and the IDE shows them: deposit, withdraw, transfer, close. There is no scattered hunt through service classes to find the relevant method.
- **The vocabulary matches the domain.** When the business person says *"a closed account cannot accept deposits"*, the developer can point at a method on `BankAccount` that says exactly that. The translation gap between English requirement and C# code shrinks.
- **The lineage is august.** This is what object orientation was *for*. Classes were invented to bundle behaviour with the data it operates on. Throwing the behaviour back out into procedural service classes — the argument goes — is to throw away the central insight of the paradigm.

This is the Domain-Driven Design position, more or less. Eric Evans's book made the case at length and persuasively. Martin Fowler's *Anaemic Domain Model* essay coined the pejorative term that has stuck to the opposing camp for twenty years.

It is a good case. I do not want it dismissed.

## The case for the anaemic model

An *anaemic* domain model — though its proponents would object to the word, and we will use it neutrally here — is one where the data lives on one set of classes and the behaviour lives on another. A `BankAccount` is a record with a balance, a status, an overdraft limit. A separate `AccountService` (or `WithdrawalHandler`, or `MoneyMover`) contains the methods that operate on accounts.

The argument runs:

- **Behaviour is easier to test in isolation.** A service class with explicit dependencies is straightforward to unit-test. A rich model with internal state and emergent collaborators is, in practice, harder to fixture.
- **Transactions belong outside the model.** A `Transfer` between two accounts is not a thing a single `BankAccount` knows about — it is a coordinating operation that touches two accounts, a journal, an event bus, and possibly an external system. Putting `Transfer` as a method on `BankAccount` forces awkward decisions about which account *owns* the operation.
- **Persistence frameworks prefer data-shaped classes.** EF Core, Dapper, and friends are at their happiest with classes that are mostly properties. The more behaviour and private state a class carries, the more hoops the persistence layer has to jump through to materialise it.
- **Immutability is easier to enforce.** A class with no behaviour can be a `readonly record`. A class with mutating methods cannot.
- **Cross-cutting concerns sit naturally in services.** Logging, authorisation, validation pipelines, retry policies — these are easier to weave around a service method than around an entity method.
- **The codebase scales differently.** A rich model accretes behaviour over years; the `BankAccount` class that started slim ends up six hundred lines long. Anaemic models accrete *handler* classes instead, which is at least a flatter form of growth.

This is the position you will find in most modern .NET codebases, often arrived at by accident rather than by principle. It is, I think, also a good case. I do not want it dismissed either.

## Where each thrives

Rich models are at their best when:

- The domain has **strong, well-understood invariants** that everyone agrees on — financial accounts, inventory levels, booking calendars.
- The behaviour set is **bounded and stable** — the operations on a `BankAccount` are not going to change much next quarter.
- The team has the **DDD literacy and discipline** to keep the model focused, refusing to let it become the dumping ground for any operation that mentions an account.
- The **persistence layer is sympathetic** — Marten over Postgres, or hand-written repositories, both happily round-trip rich objects. EF Core can be made to, with effort.

Anaemic models are at their best when:

- The domain is **largely CRUD** — most operations are create, read, update, delete, with minimal business logic.
- The system **evolves rapidly** and new operations are added all the time — easier to write a new handler than to extend an existing entity.
- The team is **larger and more heterogeneous** — flatter handler-shaped code has a shallower learning curve.
- The **persistence layer is the centre of gravity** — EF Core, large schemas, generated migrations.

You can probably see the shape of this. Neither approach is universally right. The right answer depends on what kind of domain you have, what kind of team, what kind of pressure you are under, and what kind of growth you expect.

## Where each curdles

Rich models go wrong when behaviour keeps accreting onto the entity until the class is unrecognisable. The classic failure mode is the *god entity* — `Order.cs` is fourteen hundred lines, no developer fully understands all of its methods, and changing any one of them risks breaking three others.

Anaemic models go wrong when the behaviour is scattered so widely that no one can find it. The classic failure mode is the *handler explosion* — there are forty different services that each touch `Account`, the rules about what can be done to an account are spread across all forty of them, and a perfectly reasonable change to *the rules for closing an account* requires editing eleven files.

> *A rich domain model that has spent five years successfully accreting behaviour is a thing of considerable beauty. The same model in year seven is, more often than not, a four-bedroom house with twenty-three loft conversions and a planning officer who will no longer answer the phone.*

## A note on practical middle ground

In real codebases, the line is rarely as crisp as the headline debate suggests. Most working systems sit somewhere on a spectrum:

- **Pure rich.** Behaviour entirely on entities. Services are thin orchestrators. Rare in production.
- **Hybrid favouring rich.** Core invariants on entities, cross-entity operations in domain services. Common in well-disciplined DDD shops.
- **Hybrid favouring anaemic.** Entities are mostly data with a sprinkling of methods for the most local invariants; serious behaviour in handlers. Common everywhere else.
- **Pure anaemic.** Entities are records, all behaviour in handlers. Common in transaction-script-style codebases.

The choice between the hybrids is, in my experience, more consequential than the choice between the extremes. And the choice itself often matters less than the *consistency* of the choice — a codebase that picks one and applies it everywhere is easier to work in than one that drifts between modes without noticing.

## Performance, in passing

A modest set of observations.

Rich models can be faster: they require no separate allocation for service classes, no constructor-injection chain for every operation, no mediator-pipeline overhead. The methods are calls on the object you already have.

Rich models can also be slower: lazy-loaded navigation properties that fire unexpected queries, hidden state that prevents the EF change tracker from short-circuiting, and the temptation to load whole aggregates when a projection would do.

Anaemic models pay a small per-operation allocation cost for handler instantiation (mostly mitigated by DI scoping) and a small mental tax in following the data flow, but their per-operation work is usually purely procedural and very fast.

The honest answer is that *neither approach is meaningfully faster than the other at the level of correctness most teams need to worry about*. Pick on grounds of clarity and scaling, not microseconds.

## The smells of each

**Rich-model smells:**

- An entity class over five hundred lines.
- A method on an entity that takes an `IRepository` parameter — the entity is reaching outside itself for data.
- A method on an entity that returns an `IEnumerable<>` produced by walking a navigation property that just lazy-loaded — the entity is doing I/O.
- A method on an entity called `Transfer(BankAccount other)` whose body is more orchestration than rule-enforcement.

**Anaemic-model smells:**

- A handler called `AccountManager` that contains every operation on accounts that anyone could think of.
- An entity with no validation in its constructor — you can produce an invalid `Order` whenever you like.
- Validation logic copy-pasted across three handlers that all need to check the same rule.
- A `record` with thirty properties and no method that enforces any consistency between them.

## Recap

- The rich-vs-anaemic debate is older, more ideological, and less clear-cut than its loudest participants suggest.
- Rich models put behaviour and data together; anaemic models separate them into entities and services.
- Each thrives in different conditions; each has its own characteristic failure mode at scale.
- Most real codebases are hybrids, and the *consistency* of the hybrid often matters more than the choice between them.
- Performance differences are usually too small to drive the decision.

## Onwards

The next chapter steps from the macro question of behaviour placement to a smaller, sharper, and arguably more consequential one: how you represent the small values that pepper your domain. Strings used as identifiers, decimals used as money, integers used as quantities. The trouble has a name — *primitive obsession* — and getting it right is one of the single largest correctness wins available to a working developer.
