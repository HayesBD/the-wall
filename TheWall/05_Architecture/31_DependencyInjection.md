# Chapter 31 — Dependency Injection, and How to Stop It Eating the House

Dependency injection is one of the genuinely good ideas in modern application architecture, and it has, in a great many codebases, grown into one of the genuinely frightening ones. The pattern itself is sound. The *container* — the machinery that holds the registrations and resolves the graph — is a fine tool. What goes wrong is almost never the idea and almost always the *quantity*: a container that started with twelve registrations and now has four hundred, a constructor that started with two parameters and now has nine, a startup sequence that nobody fully understands and everybody is afraid to touch.

This chapter is about keeping dependency injection in the role it is good at, and out of the role where it slowly consumes the maintainability of the application.

## What dependency injection actually is

Strip away the framework and the vocabulary, and dependency injection is a single, simple idea: *a class does not construct its own collaborators; it is given them.*

The un-injected version:

```csharp
public class OrderService
{
    private readonly EmailSender _email = new EmailSender();   // constructs its own
    private readonly OrderRepository _repo = new OrderRepository(new SqlConnection(...));
}
```

The injected version:

```csharp
public class OrderService
{
    private readonly IEmailSender _email;
    private readonly IOrderRepository _repo;

    public OrderService(IEmailSender email, IOrderRepository repo)   // is given them
    {
        _email = email;
        _repo = repo;
    }
}
```

That is the whole idea. The class declares what it needs; something else — the *container*, at the composition root — decides what to supply and hands it over. The benefit is that the collaborators can be swapped (a real `EmailSender` in production, a fake in tests), the construction logic lives in one place, and the class itself is decoupled from the concrete details of its dependencies.

This is good. I want to be clear that this is good before I spend the rest of the chapter on the ways it goes wrong.

## Service lifetimes — and the bugs in each

The DI container resolves dependencies according to their registered *lifetime*. There are three, and getting them wrong is the source of a specific and nasty class of bug.

**Singleton.** One instance for the entire application lifetime. Created once, shared by everyone. Right for stateless services, caches, configuration, and anything genuinely shared. Wrong for anything holding per-request state.

**Scoped.** One instance per scope — which, in a web application, means per HTTP request. The `DbContext` is the canonical scoped service: one per request, shared by everything handling that request, disposed when the request ends.

**Transient.** A new instance every time it is resolved. Right for lightweight, stateless services where sharing buys nothing. The default reach when you are not sure, though *scoped* is often the more correct not-sure default in a web app.

The bug that gets everyone, sooner or later, is the **captive dependency**: a scoped (or transient) service injected into a singleton.

```csharp
// Registered as singleton.
public class CacheWarmer
{
    private readonly AppDbContext _db;   // scoped — this is the bug
    public CacheWarmer(AppDbContext db) => _db = db;
}
```

The singleton is created once and lives forever. The scoped `DbContext` it captured is therefore *also* held forever — long past the request it belonged to, long past its intended disposal. The `DbContext` accumulates tracked entities, leaks memory, and eventually throws when used across threads it was never meant to span. The symptom appears far from the cause, days later, intermittently.

Modern .NET catches the most obvious cases of this at startup with scope validation, which is on by default in development. Leave it on. Treat a captive-dependency error as a design problem, not an inconvenience to be suppressed.

## Constructor over-injection — the smell that counts

The single most reliable smell in a DI-heavy codebase is the *long constructor*.

```csharp
public OrderService(
    IOrderRepository orders,
    ICustomerRepository customers,
    IEmailSender email,
    ISmsSender sms,
    IInventoryService inventory,
    IPaymentGateway payments,
    IAuditLogger audit,
    IEventBus events,
    IClock clock)
{
    // nine fields assigned ...
}
```

Nine dependencies. The class compiles, the container resolves it, the tests (with nine mocks) pass. And yet something is wrong, and the constructor is telling you exactly what: *this class does too much*. Nine collaborators is nine reasons to change, nine things to understand, nine mocks to configure in every test. The class is not a service; it is a small department.

The cure is not to hide the dependencies behind a *service aggregator* or a *facade* that injects the nine and re-exposes them — that just moves the problem. The cure is to recognise that a class needing nine collaborators is a class doing the work of three or four classes, and to split it along the seams the dependencies reveal. The order-placement logic needs the repository, the inventory, and the payment gateway. The notification logic needs the email and SMS senders. The audit logic needs the audit logger and the clock. Three classes, three or fewer dependencies each, each independently testable and independently comprehensible.

A working rule of thumb, held loosely: *more than four constructor dependencies is a prompt to look for the class hiding inside this class.*

> *A class that requires nine collaborators to do its work has not been dependency-injected so much as assembled from a kit — and like most things assembled from a kit, it will have three screws left over and a faint wobble nobody can quite locate.*

## When not to use DI at all

Not everything belongs in the container. The reflex to inject *everything* is the reflex that produces the four-hundred-registration startup.

**Pure functions and stateless utilities** do not need injecting. A `static class StringExtensions` with a `ToSlug` method is not a dependency; it is a function. Injecting an `ISlugGenerator` so it can be mocked is ceremony in search of a problem — there is nothing to mock about a pure function, and the test that calls the real one is simpler and more honest than the test that configures a fake. (We will treat this distinction properly in [Chapter 33](./33_ServicesVsUtilities.md).)

**Value objects and domain entities** are not resolved from the container. You construct a `Money` or a `Customer` directly; they are data, not services. A container that is asked to resolve domain objects has been pressed into a role it was never meant for.

**Configuration that never changes** can often be a constant or a static readonly field rather than an injected `IOptions<T>`. The options pattern is right for configuration that varies by environment or reloads at runtime; it is overkill for a value that is the same in every deployment forever.

The test for whether something belongs in the container: *does it have a dependency of its own, a lifetime that matters, or an implementation that legitimately varies?* If none of the three, it probably does not need to be a registered service.

## The composition root

The container should be configured in exactly one place — the *composition root*, near the application's entry point. This is the one place in the application that is allowed to know about both the interfaces and their concrete implementations; everywhere else depends only on the interfaces.

```csharp
// Program.cs — the composition root, and the only place wiring lives.
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
```

The discipline that makes this work — and the discipline that [Chapter 20](../03_ADifferentWay/20_BindingsWithoutAssembly.md) argued for at length — is that this list is *explicit and readable*. A new starter can read the composition root and know what the application is made of. The moment the registrations move into an assembly scan *"to save typing,"* that readability is gone and the slow slide into the four-hundred-registration mystery begins.

## The interface-per-class antipattern

A habit worth naming because it is so widespread: defining an interface for every single class, named identically to the class but with an `I` prefix, with exactly one implementation, forever.

```csharp
public interface IOrderService { /* ... */ }
public class OrderService : IOrderService { /* ... */ }
```

The justification is usually *"for testability"* or *"for loose coupling."* Examine it. An interface with exactly one implementation provides no polymorphism — there is nothing to swap. It provides testability only if you are mocking the class, and the question of whether you *should* mock a given collaborator is a real one with a frequent answer of *no* (mock the things at the system boundary — the database, the email sender, the clock; do not mock your own domain logic). The interface that exists solely so that `OrderService` can be mocked in a test of `OrderController` is often a sign that the test is testing the wrong thing.

This is not an argument against interfaces. It is an argument against the *reflexive, universal* `IFoo`/`Foo` pairing. Define an interface when there is, or genuinely will be, more than one implementation, or at a true seam (the boundary between your application and an external system). Do not define one out of habit for every class in the domain.

## Performance, in passing

DI has a runtime cost, and at the scale most applications operate it is negligible — resolving a scoped graph per request is microseconds. Two things are worth knowing nonetheless.

Transient services in a hot path allocate on every resolution. A transient service resolved a thousand times in a request is a thousand allocations; if it is genuinely stateless, a singleton would have allocated once. Match the lifetime to the actual statefulness.

The startup cost of *building* the container — which, as Chapter 20 discussed, balloons under assembly scanning — is the larger and more avoidable expense. Explicit registration keeps it small; scanning lets it grow without anyone noticing until cold starts become a problem.

## The smells

- A constructor with more than about four dependencies — the class is doing too much.
- A scoped or transient service captured by a singleton — the captive dependency bug, waiting.
- An `IFoo` with exactly one implementation `Foo`, defined reflexively rather than at a seam.
- A registration block that has become an assembly scan *"to save typing"* — readability has been traded for keystrokes.
- A pure utility function hidden behind an injected interface so it can be mocked — there was nothing to mock.
- A domain entity or value object being resolved from the container — it is data, not a service.
- A *facade* service whose only job is to hold and re-expose eight other injected services.

## Recap

- Dependency injection — *a class is given its collaborators rather than constructing them* — is a good idea worth keeping.
- Match lifetimes to statefulness; watch for the captive-dependency bug (scoped inside singleton).
- More than four constructor dependencies is a prompt to split the class.
- Not everything belongs in the container — pure functions, value objects, and fixed configuration usually do not.
- Keep the composition root explicit and readable; resist the reflexive `IFoo`/`Foo` pairing.

## Onwards

The next chapter takes a feature that a surprising number of working developers cannot, if pressed, define without gesturing vaguely at *"the pipeline"* — middleware. We will say exactly what it is, exactly what it is for, and the specific and common cases where the right answer is *"please, anything but middleware."*
