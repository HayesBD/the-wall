# Chapter 13 — On Identifiers: `Id`, `PublicId`, and the Trouble with Exposing Either

We spent a chapter arguing that domain values deserve their own types. The natural follow-up — and the one that comes up first in any real codebase — is *what about the identifier?* What about the `int Id` that EF Core generated for me five minutes after `dotnet new`? What about the `Guid Id` I dutifully put on every aggregate root because the DDD book told me to?

This chapter is, in essence, a recommendation. The recommendation is that almost every entity in your system should have *two* identifiers, not one, that they should be of different types, and that they should never, under any circumstances, leak across the boundary the other one is meant to live behind.

Bear with me.

## The trouble with exposing the internal `Id`

The internal `Id` is the integer (or, sometimes, the GUID) that your database has assigned to a row. It is the primary key. It is what foreign keys point at. It is what indexes are built around. It is, in the database's view of the world, the canonical handle for the row.

It is also, in nine out of ten applications I have ever seen, casually splashed across the public surface of the system:

```
GET /api/customers/42
GET /api/orders/1058
GET /api/products/17
```

Each of those URLs is leaking three things to the outside world, and each one is a small problem.

**It leaks the number.** A URL of `/api/customers/42` tells any observer that the system contains at least forty-two customers. If they refresh tomorrow and create their own account, they can read off the new customer count from the URL bar. For a startup trying not to advertise its early-stage scale, this is mildly embarrassing. For an enterprise application whose competitors would dearly love to know its growth curve, this is information leakage.

**It enables enumeration.** A user who can read `/api/customers/42` can, with the change of a single character, attempt `/api/customers/43`. If your authorisation checks are imperfect — and one day, somewhere in the system, they will be — that change of character is a vulnerability. Sequential integer IDs are an open invitation to the kind of bug that makes the company lawyers visit the engineering floor.

**It couples the API contract to the database internals.** The day you decide to shard the customers table across two databases, or to migrate from `int` to `long` because forty-two has become four billion, or to merge two acquired systems whose `Id` ranges overlap — the day any of these happens, every URL in your API breaks, every webhook subscription breaks, every customer-side bookmark breaks. The internal `Id` was never meant to be a stable external contract, and it makes a poor one.

> *Exposing your database primary key in a URL is the architectural equivalent of writing your front door key number on your driveway in chalk — it is technically just a small piece of metadata, and you will regret it the first time anyone walks past with a copy of the same brand of lock.*

## The trouble with exposing a `Guid` instead

The first instinct, on hearing the case above, is to replace the integer with a `Guid` and call the problem solved.

```
GET /api/customers/8f3a92b4-c1d6-4a3a-b8e7-2f5d9c1e0b88
```

This is better. It is not, by itself, sufficient.

A bare `Guid` is non-enumerable, which is the headline win, but it also has costs you should know about before adopting it everywhere.

**It is sixteen bytes, not four.** Every foreign-key column that points at it is four times the size. Every clustered index built around it has four times the leaf-page footprint. Every join is comparing sixteen-byte values rather than four-byte ones. For a small system this is invisible; for a large one it is meaningful.

**It is not sequential.** A random `Guid` inserted into a clustered index causes page splits and fragmentation. SQL Server has `NEWSEQUENTIALID()` to mitigate this; other databases have their own tricks. The default behaviour is bad.

**It is ugly.** `/api/customers/8f3a92b4-c1d6-4a3a-b8e7-2f5d9c1e0b88` is not a thing a human being can read aloud over a phone, paste into a meeting note, or include in a marketing email. There are short-form opaque identifiers — ULIDs, KSUIDs, NanoIDs — that mitigate this; we will come to them shortly.

**It still couples the API to whichever `Guid` you happened to choose.** If you ever decide to migrate from one ID scheme to another, every URL still breaks.

The deeper issue is that *the same identifier is being asked to do two different jobs*: be a fast, compact, internally-stable primary key for the database, and be a stable, opaque, externally-safe handle for the API. These are different jobs. They want different shapes.

## The pattern — two identifiers, one entity

The recommendation is straightforward. Every entity has both.

```csharp
public class Customer
{
    public int Id { get; private set; }              // internal — never crosses the wire
    public Guid PublicId { get; private set; }       // external — what the world sees

    public string Name { get; private set; } = "";
    // ...
}
```

The `Id` is what every foreign key in the database points at. It is fast, small, indexed beautifully, and it never appears in a URL, a JSON response, a webhook payload, or a log line that an outside party might see.

The `PublicId` is what every URL, API response, webhook, and customer-facing reference uses. It is opaque, non-enumerable, and stable for the lifetime of the row. It has a unique index of its own, so lookups by `PublicId` are fast, but it is not the clustered primary key.

The two identifiers are independent. The `Id` may change — though it usually does not — without breaking the public surface. The `PublicId` is guaranteed not to change. Customers, partners, and your own front-end can all hold on to it forever.

## Wiring it up in EF Core

The configuration lives in your `DbContext`, either inline in `OnModelCreating` or, better, in a dedicated `IEntityTypeConfiguration<Customer>` class (the pattern we will treat properly in [Chapter 29](../04_DataAndSets/29_DbContextConfiguration.md)). The shape is the same either way.

```csharp
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.HasKey(c => c.Id);
        builder.Property(c => c.Id).ValueGeneratedOnAdd();

        builder.Property(c => c.PublicId)
               .HasDefaultValueSql("NEWID()")   // or a Guid generator of your choice
               .ValueGeneratedOnAdd();

        builder.HasIndex(c => c.PublicId).IsUnique();

        builder.Property(c => c.Name).IsRequired().HasMaxLength(200);
    }
}
```

A few things to notice:

- **`Id` is the primary key, generated by the database.** Cheap, sequential, perfect for joins.
- **`PublicId` has a database-side default and a unique index.** It is generated automatically on insert and looked up efficiently on every external request.
- **Foreign keys (in other entities) reference `Id`, not `PublicId`.** This is the whole point. Every `Order` row carries an integer `CustomerId`, which joins back to `Customer.Id` on the four-byte axis.
- **Routes and DTOs only expose `PublicId`.** The integer `Id` is `internal` to the application layer or, if you want to be strict, never serialised at all.

## The lookup pattern

Controllers take `PublicId`. Repositories translate to entities. Internal code uses `Id` for performance-sensitive paths.

```csharp
[HttpGet("/api/customers/{publicId:guid}")]
public async Task<IActionResult> Get(Guid publicId, CancellationToken ct)
{
    var customer = await _db.Customers
        .AsNoTracking()
        .SingleOrDefaultAsync(c => c.PublicId == publicId, ct);

    return customer is null
        ? NotFound()
        : Ok(CustomerResponse.From(customer));
}
```

That is the entire pattern. The route binds a `Guid`. The lookup uses the unique index on `PublicId`. The response carries a `PublicId`, never an `Id`. The internal `Id` lives entirely on the database side, where it earns its keep doing fast joins.

For the value-object enthusiasts in the room — and we should all, after [Chapter 12](./12_PrimitiveObsession.md), be among them — `PublicId` is, of course, a perfect candidate for its own type:

```csharp
public readonly record struct CustomerPublicId(Guid Value);
```

This costs nothing at the runtime layer and makes the route signature and the JSON contract self-documenting:

```csharp
public async Task<IActionResult> Get(CustomerPublicId publicId, ...)
```

A small custom model binder and a small custom JSON converter — written once, parameterised generically — give you `CustomerPublicId`, `OrderPublicId`, `ProductPublicId`, each distinct, each non-assignable to the others.

## Variations on `PublicId`

`Guid` is the default and usually the right answer, but worth being aware of the alternatives:

- **ULID / KSUID.** Sortable, time-ordered, twenty-six-character base32 strings. Insert performance is better than random `Guid`s because they are roughly sequential. URL-readable but not friendly.
- **NanoID.** A short, configurable, URL-safe random string. Twelve to twenty characters, depending on the collision tolerance you want. Pleasant to read, pleasant in a URL, slightly more compact than a `Guid`.
- **HashIds.** A reversible encoding that turns a sequential integer into a short opaque string. Useful for migrating from sequential `Id`s without breaking URLs. Note that *reversible* is doing a lot of work in that sentence — these are not cryptographically opaque, and an attacker who knows the alphabet can recover the underlying integer.
- **Slugs.** Human-readable URL fragments — `/articles/why-records-are-the-quiet-revolution`. Layered over a `PublicId`, not a replacement for one. The slug is for SEO and human comfort; the `PublicId` is the actual handle.

Pick one and use it consistently. The worst version of this pattern is one where half the entities use `Guid` `PublicId`s and the other half use NanoIDs because three different developers had three different preferences.

## Performance, in passing

The point worth landing here, and this comes up enough that I will be slightly emphatic about it: *do not use a `Guid` as your clustered primary key*. The performance penalty is real, particularly under sustained inserts. The two-identifier pattern lets you keep the `int` (or `bigint`) as the clustered key for raw insert speed and join performance, while still exposing a non-enumerable identifier to the world.

For the public lookups, the unique index on `PublicId` is what does the work, and unique indexes on `Guid` columns are perfectly fine. They are not in the join path. They are in the *lookup* path, which fires once per request, not once per row in a query.

The net effect: external safety, internal speed, and no apologies required for either.

## The smells

- `/api/things/42` anywhere in your URL space — your internal `Id` is on tour.
- A foreign-key column typed as `uniqueidentifier` (or `uuid`) when the parent's `PublicId` is sixteen bytes and the parent's `Id` is four — you are paying the GUID join cost for no benefit.
- A `PublicId` column without a unique index — your lookup is a table scan and your idempotency is wishful thinking.
- A `Guid` primary key with no `NEWSEQUENTIALID()` or equivalent — your inserts are fragmenting the clustered index.
- A controller that takes `int id` from the route and looks it up directly — congratulations, you have a sequential enumeration vulnerability.
- A migration that changes the URL shape because the underlying ID type is being switched — the `PublicId` is precisely what would have spared you this.

## Recap

- The internal `Id` belongs to the database. It is small, fast, sequential, and never leaves.
- The `PublicId` belongs to the world. It is opaque, non-enumerable, stable, and never participates in joins.
- The two-identifier pattern keeps the internal `Id`'s performance and the `PublicId`'s safety, without compromising either.
- Configure both in `IEntityTypeConfiguration<T>`; expose only `PublicId` in routes, responses, and webhooks.
- Wrap each entity's `PublicId` in its own value type so the compiler stops you mixing them up.

## Onwards

The next chapter is the consolidated catalogue — the bestiary of model smells we have been collecting in passing through Chapters 10, 11, and 12, gathered into one place with their symptoms, their causes, and their cures. After that, [Chapter 15](./15_TheWall.md) takes the most honest chapter in Part II: the one where we look at what happens to even the most disciplined model-based codebase eighteen months after the seed funding has run out.
