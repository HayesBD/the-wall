# Chapter 12 — Primitive Obsession and Its Discontents

There is a particular kind of bug that I have, over my career, watched land in production more times than I can count. It always looks slightly different on the surface. Underneath, it is always the same.

A `string` is passed in the wrong slot. A `decimal` representing one thing is added to a `decimal` representing another. An `int` representing a percentage finds itself stored in a column meant for a basis-point value. The compiler does not complain. The tests do not catch it, because the test fixtures use values that happen to be in the same range. The bug ships, lives quietly for some weeks, and is eventually noticed by a customer when their invoice arrives in the wrong currency or their booking is for the wrong year.

The disease is called **primitive obsession** — the habit of representing every value in the domain with one of the small handful of types the language gives us for free. The cure is a discipline so simple that it sounds, at first description, faintly ridiculous: *if a value has rules, give it a type.*

## What primitive obsession looks like

Here is a method signature I have seen, in various forms, in roughly every large codebase I have ever worked on.

```csharp
public Task TransferAsync(
    string sourceAccountId,
    string targetAccountId,
    decimal amount,
    string currency,
    string idempotencyKey,
    int retryCount,
    long timestamp)
```

Seven parameters. Three of them are strings. Two of them are numbers. One of them is a number that is really an enum. One of them is a number that is really a timestamp.

Question for the reader: which of these calls compiles?

```csharp
// Call A
await TransferAsync(account1.Id, account2.Id, 100.00m, "GBP", "txn-001", 3,
    DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());

// Call B
await TransferAsync(account2.Id, account1.Id, 100.00m, "USD", "txn-001", 3,
    DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());

// Call C
await TransferAsync(account1.Id, "txn-001", 100.00m, "GBP", account2.Id, 3,
    DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());
```

The answer is *all three of them*. The first two are different operations on the same accounts. The third is nonsense — the source account ID has been swapped with the idempotency key, the target account ID has been swapped with the currency — but the compiler is satisfied, because every slot is a string and every string is interchangeable as far as the type system is concerned.

This is primitive obsession. It is the most preventable category of bug in modern business software, and it is preventable for the price of a slightly larger type vocabulary.

## The cure — value objects

A value object is a small type whose only purpose is to *be* a particular kind of value, with the rules that go along with it. Records and `readonly record struct`s make these almost free to write.

```csharp
public readonly record struct AccountId(Guid Value);
public readonly record struct IdempotencyKey(string Value);
public readonly record struct Currency(string Code);
public readonly record struct Money(decimal Amount, Currency Currency);
public readonly record struct RetryCount(int Value);
```

Now the same signature looks like this:

```csharp
public Task TransferAsync(
    AccountId source,
    AccountId target,
    Money amount,
    IdempotencyKey key,
    RetryCount retries,
    DateTimeOffset at)
```

And the nonsense call no longer compiles. The compiler will refuse to pass an `IdempotencyKey` where an `AccountId` is expected. It will refuse to pass a `decimal` where a `Money` is expected. It will refuse to add a `Money` denominated in pounds to a `Money` denominated in dollars, if you wrote the `+` operator on `Money` properly.

The bug class evaporates. Not *is less common*. Evaporates. The compiler simply will not allow the program to express it.

## Putting the rules inside the type

The other half of the value-object pattern is that the rules about what makes a value *valid* live inside the type, enforced at construction.

```csharp
public readonly record struct Currency
{
    public string Code { get; }
    public Currency(string code)
    {
        if (code is null || code.Length != 3 || !code.All(char.IsAsciiLetterUpper))
            throw new ArgumentException("Currency must be a three-letter ISO code.", nameof(code));
        Code = code;
    }
}
```

Now no `Currency` exists in your program with a value other than three uppercase ASCII letters. The check happens once, at the moment a `Currency` is constructed, and the rest of the codebase can treat any `Currency` it receives as already-valid. No more defensive `if (!IsValidCurrency(code))` scattered through twenty methods. No more guard clauses in five different services that all enforce the same rule slightly differently. One type. One rule. One place.

This is one of the quietest correctness wins available to a working developer, and it gets quieter still in proportion to how widely the value object is used.

## The "but it's so much code" objection

The objection I hear most often, when I propose this in a code review, is that it is *so much code*. Six small types where there used to be one method parameter list. Six constructors. Six equality implementations. Six `ToString`s.

Records eat that objection in one bite. A `readonly record struct AccountId(Guid Value)` is one line. The compiler emits the equality, the hash code, the `ToString`, the deconstruction, the constructor, the lot. Adding a value object now costs almost nothing.

The objection has a second form, slightly more sophisticated: *we will lose the friendliness of working with raw primitives*. We will not be able to call `.ToString("X")` on an `AccountId`. We will not be able to add two `Money`s with `+`. We will, in short, have made things harder.

The first concern is easily addressed — a value object can expose whatever methods make sense for its domain. An `AccountId` does not need to support hex formatting; it needs to support equality, comparison, and serialisation. A `Money` *does* need `+` and `-`, and writing those operators on a record is the work of a few lines.

The second concern, in honesty, is the real one. Value objects do add a small ceremonial cost at the boundaries of your system — request validators, JSON converters, database type mappings. The cost is real. It is also a one-time cost per value type, paid in well-defined places, in exchange for a permanent correctness gain across the entire codebase. Most teams that have made the trade have not regretted it.

## On identifiers, briefly

The most common and most consequential value-object opportunity in any application is the identifier. `int Id`, `Guid Id`, and `string Id` are the three most-overused types in the language, and almost every entity in your domain wants its own.

```csharp
public readonly record struct CustomerId(Guid Value);
public readonly record struct OrderId(Guid Value);
public readonly record struct ProductId(Guid Value);
```

A `CustomerId` is no longer assignable to an `OrderId`. The bug where a customer ID is passed to a method expecting an order ID — a bug I have personally introduced into production, and which is no fun to find — is now a compile error.

The next chapter takes this further, covering the *exposed* identifier (the `PublicId` you serialise) versus the *internal* identifier (the `Id` your database uses), and the lovely pattern that lets `OnModelCreating` in your `DbContext` do the translation between them once and for all.

> *Representing a customer ID as a `string` because *"a string is what it is"* is rather like representing a hand grenade as *"a small heavy object"* — technically correct, and pleasingly compact, right up to the moment you need to know rather more about the object than its dimensions.*

## Performance, in passing

The objection that value objects cost performance has, in modern C#, a one-word answer: `struct`. A `readonly record struct AccountId(Guid Value)` is the same sixteen bytes as the underlying `Guid`. There is no heap allocation. There is no copy beyond what you would do with the raw `Guid`. The runtime treats them identically.

For value objects that *do* allocate — anything wrapping a `string` or another reference type, in record-class form — the cost is one small heap allocation at construction and one reference per pass thereafter. For an `IdempotencyKey` constructed once at request entry and passed through ten methods, this is unmeasurable. For an `AccountId` constructed a million times in a tight loop, prefer the struct form.

A small bonus: by giving every domain value its own type, you also give EF Core, JSON serialisers, and validation libraries something concrete to bind to. Value converters, custom JSON converters, and FluentValidation rules can all be written once per type and reused everywhere that type appears. The boundary code becomes more boilerplate-heavy; the application code becomes much cleaner.

## The smells

- Method signatures with three or more `string` parameters of which one is an identifier, one is a name, and one is a code.
- Validation logic for the same rule (currency-code format, email format, postcode format) repeated in three or more files.
- A `decimal` parameter named `amount` with no currency anywhere nearby.
- Foreign-key columns that store an `int` and the entity-side property is also an `int` — the wrong customer being assigned to the wrong order is one swapped argument away.
- Comments next to method parameters explaining what unit or what range the primitive is in. The comment is the type telling you it wanted to be a type.

## Recap

- Primitive obsession is the habit of representing every domain value as a `string`, `int`, or `decimal`.
- It is the most preventable class of bug in modern business software.
- The cure is value objects, and modern C# makes them almost free with `readonly record struct`.
- Put validation rules inside the value object's constructor so the rule lives in one place.
- The performance cost, for struct-based value objects, is nil.

## Onwards

The next chapter takes the most consequential value-object opportunity in any application — the identifier — and looks at the `Id` versus `PublicId` pattern that keeps your database internals out of your URLs, your API responses, and your customers' inboxes. It is one of the small disciplines that, once adopted, you will never want to be without.
