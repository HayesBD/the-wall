# Chapter 28 — Cardinality and Relationships: Modelling the Shape of the World

The previous chapter was about asking the right questions of the data. This chapter is about laying the data out in the shape that lets the right questions be asked at all.

The four cardinalities — one-to-one, one-to-many, many-to-one, many-to-many — are, on paper, the most thoroughly settled concept in relational design. Every introductory database textbook covers them. Every ORM supports them. Every database engineer can rattle them off in their sleep. And yet, in roughly every codebase I have ever audited, at least one of them is being modelled wrong in a way that costs the team something.

This chapter is about which one is the right reach in which situation, where the common mistakes hide, and how to wire each in EF Core in a way that does not regret itself eighteen months later.

## The four shapes, in plain English

**One-to-many.** A `Customer` has many `Order`s. An `Order` has one `Customer`. This is the canonical, default, most-common shape in business software. The foreign key lives on the *many* side — `Orders.CustomerId` points at `Customers.Id`. Navigation properties may exist on one side, both sides, or neither.

**Many-to-one.** The same shape from the other side. `Order` belongs to one `Customer`. There is no structural difference; it is the same relationship described from a different vantage point. The same foreign key.

**One-to-one.** Every `User` has exactly one `UserSettings`. Every `Order` has exactly one `OrderShipment`. The foreign key lives on the *dependent* side, with a unique constraint to enforce the *exactly one* part. This is the cardinality most commonly modelled wrong, because true 1:1 relationships are rarer than people think.

**Many-to-many.** A `Customer` may have many `Tag`s, and each `Tag` may apply to many `Customer`s. Neither side owns the other. Implemented either as a *skip navigation* (EF Core 5+ syntactic sugar, with an implicit join table) or as an *explicit join entity* (`CustomerTag`), with a foreign key to each side and, usually, additional columns of its own.

## The wrong shape, three common ways

Before recommending which shape goes where, the failure modes worth knowing.

**The "1:1" that is really 1:N.** *Every customer has one billing address.* Sounds like a 1:1. Modelled as a 1:1 with a unique constraint on `BillingAddresses.CustomerId`. Two years later, the business asks to *historically track* billing address changes; or to support *multiple billing addresses* for international customers; or to *share* a billing address between related accounts. The 1:1 was a temporary 1:N pretending to be a 1:1.

The rule of thumb: if there is any plausible future where a parent might have more than one of these, model it as 1:N from the start. The cost of the extra row-per-link is small. The cost of the migration from 1:1 to 1:N — touching the foreign key, the schema, the entity class, the configurations, the API, the validations — is large.

**The M:N that should be three 1:N's.** *Customers have orders, orders have products.* The naive reading is that customers and products are in a many-to-many relationship via orders. The correct reading is that there are three separate one-to-many relationships: `Customer 1:N Order`, `Order 1:N OrderLine`, `Product 1:N OrderLine`. The `OrderLine` carries the *additional facts* — quantity, unit price, line total — that the M:N abstraction would have erased.

The rule of thumb: the moment a relationship grows even one attribute of its own (a date, a quantity, a role, a status), it is not really a many-to-many relationship; it is an entity in its own right that happens to refer to two other entities.

**The M:N that should be 1:N in disguise.** *A `User` may belong to many `Role`s.* On the surface, M:N. In practice, almost every system that models this finds itself wanting a single *current* role per user for billing or display, with the others being historical or capability-style. The 1:N (with effective dates, or with a *primary* flag) is what was wanted; the M:N is an attractive nuisance.

## Modelling many-to-many — the two ways

When a many-to-many is genuinely the right shape, EF Core gives you two ways to express it.

**Skip navigation (EF Core 5+).** The natural-language form. The join table is implicit.

```csharp
public class Customer
{
    public int Id { get; set; }
    public ICollection<Tag> Tags { get; set; } = new List<Tag>();
}

public class Tag
{
    public int Id { get; set; }
    public ICollection<Customer> Customers { get; set; } = new List<Customer>();
}
```

EF Core, on `OnModelCreating`, creates a `CustomerTag` join table behind the scenes, with the appropriate foreign keys. You can navigate from a customer to its tags, or from a tag to its customers, without ever referring to the join table directly.

**Explicit join entity.** The join table is its own type, with a foreign key to each side and any additional columns it needs.

```csharp
public class Customer
{
    public int Id { get; set; }
    public ICollection<CustomerTag> CustomerTags { get; set; } = new List<CustomerTag>();
}

public class Tag
{
    public int Id { get; set; }
    public ICollection<CustomerTag> CustomerTags { get; set; } = new List<CustomerTag>();
}

public class CustomerTag
{
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    public int TagId { get; set; }
    public Tag Tag { get; set; } = null!;
    public DateTimeOffset AppliedAt { get; set; }
    public UserId AppliedBy { get; set; }
}
```

The explicit form is wordier, more files, more code. It is also *invariably* the right reach the moment the relationship has any attribute of its own — *who* applied the tag, *when*, *why*. The skip navigation, in those cases, has nowhere to put the data.

My recommendation, fairly strongly held: *reach for the explicit join entity by default*. The skip-navigation form is convenient, and it is the right reach for relationships that genuinely carry no data of their own — *user belongs to role*, with no metadata about the assignment — but those relationships are rarer than the alternative. Designing for the explicit form keeps the door open; designing for the skip-navigation form closes it.

## Required, optional, and the meaning of `null`

Each end of a relationship is either *required* or *optional*, and the distinction is more consequential than people remember.

A *required* end means *"there must be one of these"*. A required parent on the many side of a 1:N means every child must have a parent. The foreign key is non-nullable. A child without a parent is a constraint violation that the database refuses.

An *optional* end means *"there might be one of these"*. The foreign key is nullable. The child can exist without a parent.

The wrong choice in either direction is the source of a recurring class of bug. Required when it should be optional means *"every booking must have a confirmed user"* — fine until guest checkout arrives. Optional when it should be required means *"some orders have no customer"* — fine in the abstract, a disaster the first time a customer-service ticket asks who placed the orphan order.

In EF Core, the cardinality is configured explicitly:

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId)
    .IsRequired();   // explicit and visible
```

The `IsRequired()` is the difference between a non-nullable and a nullable foreign key column. *Be deliberate about it.* The default behaviour depends on the nullability of the property type, which is a perfectly reasonable convention until the day you change a non-nullable type to a nullable one for an unrelated reason and silently change the database schema in the process.

## Navigation properties — when and on which side

A navigation property is the C# representation of a relationship — `customer.Orders` to walk from a customer to their orders, or `order.Customer` to walk back. The foreign key is what makes the database join work; the navigation property is sugar that makes the C# read naturally.

You have three choices for each relationship: navigation on one side, navigation on both sides, or navigation on neither.

**Both sides** — sometimes called *bidirectional navigation* — is convenient but expensive. Lazy loading fires on either side. Serialisers must be configured to avoid cycles. EF Core's change tracker must track both ends. For relationships where the application genuinely walks in both directions, the convenience may be worth it.

**One side** is the right reach in most cases. The application walks one way (`customer.Orders`); the other direction is achievable when needed via a query (`db.Customers.Single(c => c.Orders.Any(o => o.Id == orderId))`). The model is leaner, the tracker is lighter, the cycles do not arise.

**Neither side** is a perfectly defensible choice for relationships you only ever query relationally — *"join customers to orders"* — without ever navigating an in-memory graph. The foreign key column carries the relationship; the application code reaches for `Where` and `Join` instead of dot-walking.

The smell: *bidirectional navigation that the application only ever uses in one direction*. This is the most common cardinality-related smell in EF Core codebases and almost always worth removing.

## Owned types — value objects in entity clothing

A close relative of relationships, worth a short section because the boundary is fuzzy. An *owned type* is a value object whose properties are mapped *into the parent entity's table* rather than into a separate table.

```csharp
modelBuilder.Entity<Customer>().OwnsOne(c => c.BillingAddress);
```

`BillingAddress` is now a property on `Customer`, but the address's fields — `Street`, `City`, `Postcode` — become columns on the `Customers` table directly. There is no `Addresses` table, no foreign key, no navigation in the relational sense. The address has no identity of its own; it exists only as part of its parent.

This is the right reach for genuine value objects — money, dates, postal addresses, GPS coordinates — that *belong* to a single parent and have no meaning apart from it. It is the wrong reach for things that might be shared (an address used by both billing and shipping), things that might need to be queried independently (find all customers in postcode SW1A 1AA), or things with their own lifecycle.

## Foreign-key conventions, briefly

EF Core's conventions for foreign keys are good and worth following.

- A property named `CustomerId` on `Order`, where `Customer` is the navigation, is assumed to be the foreign key.
- The foreign key type matches the primary key type of the principal.
- The foreign key is nullable if and only if the navigation property is nullable.

Where the conventions do not match your intent, override explicitly in `IEntityTypeConfiguration<T>` (the pattern we will treat properly in [Chapter 29](./29_DbContextConfiguration.md)) rather than fighting the convention from a thousand small attributes scattered across the entity class.

## Performance, in passing

A few small notes that recur across real codebases.

Joins on the integer primary key are fast. Joins on the public `Guid` are slower. The two-identifier pattern from [Chapter 13](../02_TheModelQuestion/13_OnIdentifiers.md) is the right reach precisely because it keeps the integer for joins and the Guid for the outside world.

Many-to-many with a skip navigation is, for read paths, marginally faster than the explicit join entity — fewer indirections, less data on the wire. The difference is rarely meaningful; the explicit form's correctness benefits dominate.

Required relationships permit the database to produce a tighter join plan than optional ones. Where a relationship is genuinely required, marking it so produces both better data integrity and (in some cases, on some databases) better query performance.

> *Modelling every plausibly-optional relationship as optional, "just in case", is the database equivalent of refusing to commit to a dinner invitation in case something more interesting comes up — the host plans for both outcomes, the cooking takes twice as long, and nobody enjoys themselves quite as much as they should.*

## The smells

- A 1:1 relationship with no clear reason it could not become 1:N.
- A many-to-many via skip navigation that has been augmented, post hoc, with a `Notes` column on a fake middle entity to carry data the relationship now needs.
- Bidirectional navigation properties on an entity pair where one direction is never used.
- A foreign key without an index — every `JOIN` on it is a full scan.
- An optional foreign key whose business meaning is *"required, but we did not want to refactor the existing rows"* — the schema is now lying about the data.
- A *"link table"* with three or more foreign keys and no clear name — you have an entity, treat it as one.
- A many-to-many between entities whose relationship clearly carries data, with the data smuggled in via a `Dictionary<...>` somewhere in C# code.

## Recap

- The four cardinalities are 1:1, 1:N, N:1, M:N. The N:1 is the same shape as 1:N from the other side.
- True 1:1 relationships are rarer than people model them. Prefer 1:N unless you are certain.
- Many-to-many with attributes of its own is not really M:N — it is an entity. Use an explicit join entity.
- Required vs optional is meaningful. Be deliberate.
- Navigation properties are sugar; bidirectional sugar is expensive.
- Owned types are the right reach for genuine value objects with no independent identity.

## Onwards

The next chapter is the practical follow-up. We have spent four chapters on the *shape* of data — collections, sets, queries, relationships. The next chapter is about where the *configuration* of all of this belongs in an EF Core codebase: not on the entity, not in the controller, not scattered across `OnModelCreating`, but in dedicated per-entity configuration classes that turn the persistence layer into something a thoughtful engineer can read in a single sitting. It is one of those small disciplines that, once adopted, you will never want to be without.
