# Chapter 33 — Services vs Utilities: Knowing the Difference

Here is a distinction that sounds almost too small to merit a chapter, and that — once it has lodged properly in your thinking — quietly eliminates a remarkable amount of daily architectural dithering. It is the distinction between a *service* and a *utility*.

The dithering it eliminates looks like this. A developer needs to write some code that, say, formats a postal address into a single display string. Where does it go? Should it be a class? Should it have an interface? Should it be registered in the container? Should it be injected? Should it be mockable? Each of these questions gets a few minutes of thought, a code-review comment, perhaps a small disagreement. Multiply by every small piece of logic in a large codebase and the cumulative cost is substantial — and almost all of it is avoidable, because almost all of it has a clear answer once you can tell a service from a utility.

## The two categories

**A service** is a thing that *does work on behalf of the application, using collaborators it depends on*. It talks to a database, sends an email, calls an external API, publishes an event, reads the clock. It has *dependencies*. It often has a *lifetime* that matters — it might be scoped to a request, or a singleton shared across the application. It is registered in the container and injected where needed.

**A utility** is a thing that *transforms its inputs into outputs and depends on nothing*. It formats a string, calculates a checksum, parses a date, slugifies a title, validates a postcode against a regex. It has *no dependencies*. It has *no state*. It has *no lifetime* — there is nothing to keep alive between calls. It is, in essence, a *pure function*, and the right home for it is a static method.

The entire distinction comes down to three linked questions:

- Does it have **dependencies**? Service if yes, utility if no.
- Does it hold **state** or have a **lifetime that matters**? Service if yes, utility if no.
- Does its **implementation legitimately vary** (real vs fake, one provider vs another)? Service if yes, utility if no.

If the answer to all three is *no*, you have a utility, and you should stop dithering and write a static method.

## The utility, done right

```csharp
public static class AddressFormatter
{
    public static string ToSingleLine(Address address) =>
        string.Join(", ", new[]
        {
            address.Line1,
            address.Line2,
            address.City,
            address.Postcode
        }.Where(part => !string.IsNullOrWhiteSpace(part)));
}
```

No dependencies. No state. No lifetime. No interface. Not in the container. Called directly: `AddressFormatter.ToSingleLine(customer.Address)`.

The temptation — and I have watched it play out in code review more times than I can count — is to wrap this in an `IAddressFormatter` interface, register it as a service, and inject it everywhere it is used, *"so it can be mocked."* Resist. There is nothing to mock. The function is deterministic: the same address always produces the same string. A test that calls the real `ToSingleLine` and asserts the output is simpler, faster, and more honest than a test that configures a `Mock<IAddressFormatter>` to return a canned string — the latter tests nothing except that the mock was configured.

## The service, done right

```csharp
public class OrderConfirmationService
{
    private readonly IEmailSender _email;
    private readonly IClock _clock;

    public OrderConfirmationService(IEmailSender email, IClock clock)
    {
        _email = email;
        _clock = clock;
    }

    public async Task ConfirmAsync(Order order)
    {
        var sentAt = _clock.UtcNow;
        await _email.SendAsync(order.CustomerEmail, "Order confirmed",
            $"Your order {order.Number} was confirmed at {sentAt:u}.");
    }
}
```

Dependencies (`IEmailSender`, `IClock`). The implementations legitimately vary — a real SMTP sender in production, a fake in tests; a system clock in production, a fixed clock in tests. *Here* the interfaces earn their keep, *here* the injection is right, *here* the container registration belongs. The test for this class genuinely wants the fakes, because the real email sender would send real email and the real clock would make the assertion non-deterministic.

The contrast with the utility is the whole lesson. The utility's collaborators do not vary, so it has none, so it is a static function. The service's collaborators vary, so they are injected, so it is a registered class. The shape follows from the nature of the thing.

## The clock — the canonical boundary case

The clock deserves a paragraph because it is the example that most cleanly illustrates where the line sits.

*"What time is it?"* sounds like a pure function. It is not. `DateTime.UtcNow` returns a different answer every time it is called — it is the very definition of *non-deterministic* — and code that calls it directly is code that cannot be tested deterministically. *"Confirm the order and record that it happened at the current time"* becomes untestable the moment *the current time* is whatever the wall clock says during the test run.

So the clock is a *service*: `IClock` with a `UtcNow` property, a `SystemClock` implementation for production, a `FixedClock` for tests. It has no dependencies of its own, which makes it look utility-shaped, but it has the property that disqualifies a utility — *its output is not determined by its inputs*. That single property pushes it firmly onto the service side of the line. The same logic applies to random-number generation, to anything reading the file system, to anything reading the environment.

The refined test, then: a utility is a *pure* function — same inputs, same outputs, no observation of the outside world. The moment a thing observes the outside world (the clock, the filesystem, the network, a random source), it is a service, however few constructor parameters it has.

## When a utility grows up

Utilities sometimes acquire a dependency and, in doing so, become services. This is fine and worth recognising when it happens.

Consider a `CurrencyFormatter.Format(Money money)` that starts life as a pure utility — given a money, produce a display string using the currency's standard format. Then a requirement arrives: the format should respect the *current user's locale*. Now the function needs to know the locale, which it must read from somewhere — an `IUserContext`, an `ICurrentCulture`. It has acquired a dependency. It has crossed the line. It is now a service.

The mistake is to fight this — to keep it a static method and reach for a `static` mutable field or a `ThreadLocal` to smuggle the locale in. That way lies hidden global state, the thing [Chapter 17](../03_ADifferentWay/17_StatelessRevelation.md) spent a chapter warning about. When a utility acquires a genuine dependency, let it become a service gracefully: give it a constructor, register it, inject it. The promotion is not a failure; it is the shape correctly following the changed nature of the thing.

## The "service" that should never have been one

The reverse mistake is more common and more damaging: the *utility wearing a service's clothes*. A class with an interface, a container registration, and an injection site, whose implementation has no dependencies, no state, and no variation — a pure function dressed up as a service for no reason anyone can now recall.

```csharp
// Over-engineered: a pure function pretending to be a service.
public interface ISlugGenerator { string Generate(string title); }
public class SlugGenerator : ISlugGenerator
{
    public string Generate(string title) =>
        title.ToLowerInvariant().Replace(' ', '-');   // no dependencies, no state
}
```

Three artefacts — interface, class, registration — and an injection site in every consumer, all in service of a function that could have been `Slug.From(title)`. The cost is not just the ceremony; it is that every consumer now has one more constructor parameter, one more thing in its dependency graph, one more mock in its tests, all for a deterministic string transformation that needed none of it.

The cure is demotion: delete the interface, delete the registration, make it a static method, and call it directly. Every consumer's constructor gets shorter. Every consumer's test gets simpler. Nothing of value is lost, because there was nothing to mock in the first place.

## Performance, in passing

A small but real consideration. A static utility method has no allocation, no resolution, no indirection — it is a direct call, and the JIT will often inline it. A service, by contrast, is resolved from the container (an allocation, or at least a dictionary lookup), held as a field, and called through an interface (a virtual dispatch the JIT usually cannot inline).

For the vast majority of code, this difference is unmeasurable and should not drive the decision — correctness and clarity drive it. But it is worth knowing that the over-engineered service form is not only more ceremonious than the utility; it is also, very slightly, slower, on every call, forever. When the thing genuinely is a pure function, the static method is both the cleaner *and* the faster choice. As is so often the case in this series, the two go together.

> *Wrapping a pure function in an injectable service is rather like hiring a chauffeur to operate your electric toothbrush — the job gets done, a great deal of process now surrounds it, and you find yourself making polite conversation at moments you would really rather not.*

## The smells

- An interface with one implementation whose methods take inputs and return outputs with no dependencies — a utility dressed as a service.
- A `static` class with a `static` mutable field used to smuggle in what should have been a constructor dependency — a service refusing to admit it is one.
- A constructor parameter that is only ever used to call one deterministic method — that collaborator wanted to be a static call.
- Direct calls to `DateTime.UtcNow` / `Guid.NewGuid()` / `File.ReadAllText` inside otherwise-testable logic — the outside world has been observed without a seam, and the logic is now hard to test.
- A test that mocks a pure-function collaborator and asserts only that the mock returned its canned value — the mock is testing itself.

## Recap

- A utility is a pure function: no dependencies, no state, no lifetime, deterministic. Make it a static method.
- A service does work using collaborators that vary, or observes the outside world. Give it a constructor, register it, inject it.
- The clock, randomness, the filesystem, and the network are services even with no dependencies, because they are not deterministic.
- A utility that acquires a genuine dependency should become a service gracefully — do not smuggle the dependency in via static state.
- A pure function dressed as a service is ceremony, extra constructor parameters, and a slightly slower call, all for nothing.

## Onwards

The final chapter of this Part steps back from the individual pieces — services, utilities, middleware, dependencies — and asks how they fit together into a whole that a person can hold in their head. Module boundaries, the shape of a healthy folder structure, the difference between the public surface and the internal scaffolding, and the quiet discipline of putting things in the place the next reader will look for them first.
