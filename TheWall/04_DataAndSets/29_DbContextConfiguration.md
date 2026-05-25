# Chapter 29 — The DbContext Configuration Pattern: Where Persistence Concerns Belong

We have, several times across this series, gestured at a pattern and promised to treat it properly later. The model-binding hookup in [Chapter 13](../02_TheModelQuestion/13_OnIdentifiers.md). The value converter for value objects in [Chapter 12](../02_TheModelQuestion/12_PrimitiveObsession.md). The relationship configuration in [Chapter 28](./28_Cardinality.md). *Later* has arrived. This chapter is about where all of that configuration actually lives, and the strong recommendation that it lives in one specific, slightly unglamorous place.

The recommendation, stated up front: *every entity's persistence configuration belongs in its own `IEntityTypeConfiguration<T>` class, and nowhere else.* Not on the entity. Not in the controller. Not scattered through a four-thousand-line `OnModelCreating`. One file per entity, containing the entire story of how that entity maps to the database.

Let me make the case.

## The three ways to configure EF Core

EF Core gives you three mechanisms for telling it how your entities map to tables. They are not equally good.

**Data annotations.** Attributes on the entity's properties.

```csharp
public class Customer
{
    [Key]
    public int Id { get; set; }

    [Required, MaxLength(200)]
    public string Name { get; set; } = "";

    [Column(TypeName = "decimal(18,2)")]
    public decimal LifetimeSpend { get; set; }
}
```

Convenient. Co-located with the property. Also: persistence concerns smeared directly onto the domain model, in violation of everything we argued in [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md) about keeping the four masters apart. The `[Column(TypeName = ...)]` is a database concern living on a class that, ideally, would not know it is ever stored in a database at all. Annotations are also limited — they cannot express the more sophisticated configuration (value converters, query filters, complex keys) that real applications need.

**Fluent configuration in `OnModelCreating`.** All configuration in one giant method.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().HasKey(c => c.Id);
    modelBuilder.Entity<Customer>().Property(c => c.Name).IsRequired().HasMaxLength(200);
    modelBuilder.Entity<Order>().HasKey(o => o.Id);
    // ... four thousand more lines ...
}
```

The full expressive power of the fluent API. Also: a single method that grows without bound, in which the configuration for `Customer` is interleaved with the configuration for forty other entities, and in which finding *"everything about how `Order` is mapped"* requires a scroll and a prayer.

**`IEntityTypeConfiguration<T>` — one class per entity.** The full fluent power, partitioned by entity.

```csharp
public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.HasKey(c => c.Id);
        builder.Property(c => c.Id).ValueGeneratedOnAdd();

        builder.Property(c => c.PublicId).ValueGeneratedOnAdd();
        builder.HasIndex(c => c.PublicId).IsUnique();

        builder.Property(c => c.Name).IsRequired().HasMaxLength(200);

        builder.HasMany(c => c.Orders)
               .WithOne(o => o.Customer)
               .HasForeignKey(o => o.CustomerId)
               .IsRequired();
    }
}
```

The full power of the fluent API. The entire persistence story of `Customer` in one file called `CustomerConfiguration.cs`. The `OnModelCreating` reduced to a single line that discovers all the configuration classes automatically:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
    => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
```

This is the pattern I am recommending. Let me say why it wins.

## Why per-entity configuration wins

**Discoverability.** *"Where is `Order` configured?"* has exactly one answer: `OrderConfiguration.cs`. Not "somewhere in the 4,000-line `OnModelCreating`." Not "partly on the entity as attributes and partly in the fluent block." One file, one entity, one place.

**Testability.** A configuration class is a plain object with a single method that takes a builder. You can construct one, invoke `Configure` against a test builder, and assert that the configuration is what you expect. The giant-`OnModelCreating` approach offers no such seam.

**Separation.** The domain entity stays clean. `Customer` is a domain model with no `[Column]`, no `[Key]`, no `[MaxLength]`, no knowledge that it is ever persisted. The persistence concerns live in `CustomerConfiguration`, which is allowed to know about databases because that is its entire job. The four masters of [Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md) stay apart.

**Reuse.** Shared conventions — every entity has a `PublicId` with a unique index, every entity has audit columns — become base configuration classes or extension methods, applied consistently without copy-paste.

```csharp
public static class ConfigurationExtensions
{
    public static void ConfigurePublicId<T>(this EntityTypeBuilder<T> builder)
        where T : class, IHasPublicId
    {
        builder.Property(e => e.PublicId).ValueGeneratedOnAdd();
        builder.HasIndex(e => e.PublicId).IsUnique();
    }
}
```

## `ApplyConfigurationsFromAssembly` — and its one caveat

The single line that wires it all up:

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
```

This scans the assembly for every `IEntityTypeConfiguration<T>` implementation and applies it. It is, you will notice, a form of assembly scanning — the very thing I was rude about in [Chapter 20](../03_ADifferentWay/20_BindingsWithoutAssembly.md).

I will own the apparent contradiction. The difference is one of *blast radius*. The assembly scan in Chapter 20 was for the application's entire service graph — hundreds of registrations, runtime-resolved, opaque, slow. This scan is for the configuration classes of a single `DbContext`, a much smaller and more uniform set, applied once at model-building time, which EF Core caches for the lifetime of the application. The cost is a one-time startup expense measured in milliseconds, and the opacity is bounded — every match is an `IEntityTypeConfiguration<T>`, a single well-known shape, easy to find with the IDE.

If you prefer to be explicit — and there is a respectable argument for it — you can register each configuration by hand:

```csharp
modelBuilder.ApplyConfiguration(new CustomerConfiguration());
modelBuilder.ApplyConfiguration(new OrderConfiguration());
// ...
```

Both are defensible. The assembly scan is the common choice and the cost is genuinely small here; the explicit list is the choice for teams who want the registration fully visible. Pick one; do not mix.

## The model-binding hookup, delivered

Chapter 13 promised that the translation between the public `Guid` identifier and the internal integer key could be made to disappear from the application via configuration. Here is where it lives.

The `PublicId` is configured with its unique index in the entity's configuration class, exactly as shown above. The lookup-by-public-id pattern then lives in a small, generic repository method or a custom model binder, configured once:

```csharp
public class PublicIdModelBinder<TEntity> : IModelBinder
    where TEntity : class, IHasPublicId
{
    public async Task BindModelAsync(ModelBindingContext context)
    {
        var raw = context.ValueProvider.GetValue(context.ModelName).FirstValue;
        if (!Guid.TryParse(raw, out var publicId)) { return; }   // fail binding

        var db = context.HttpContext.RequestServices.GetRequiredService<AppDbContext>();
        var entity = await db.Set<TEntity>()
            .AsNoTracking()
            .SingleOrDefaultAsync(e => e.PublicId == publicId);

        context.Result = entity is null
            ? ModelBindingResult.Failed()
            : ModelBindingResult.Success(entity);
    }
}
```

Registered once, generically, the binder means a controller action can take the entity directly:

```csharp
[HttpGet("/api/customers/{customer:guid}")]
public IActionResult Get(Customer customer)   // bound by PublicId, behind the scenes
    => Ok(CustomerResponse.From(customer));
```

The controller never sees the internal `Id`. The translation happens once, in infrastructure code, configured in one place. This is the *"makes the whole concern disappear"* property promised three Parts ago.

## Value converters — the value-object bridge

Chapter 12 argued for value objects — `readonly record struct CustomerId(Guid Value)` and the like. EF Core does not, by default, know how to store one. The value converter is the bridge, and it belongs in the configuration class.

```csharp
builder.Property(c => c.Id)
       .HasConversion(
           id => id.Value,                   // to database
           value => new CustomerId(value));   // from database
```

For value objects used widely, register the converter globally so it applies everywhere the type appears:

```csharp
configurationBuilder
    .Properties<CustomerId>()
    .HaveConversion<CustomerIdConverter>();
```

This is the *boundary code* that Chapter 12 acknowledged value objects require. It is written once per value type, in a well-defined place, and it is the price of the compile-time safety that value objects buy everywhere else.

## Query filters — cross-cutting concerns at the data layer

Soft delete and multi-tenancy — two of the cross-cutting concerns we discussed at length in the ECS Part — have a clean expression in EF Core via *global query filters*, configured per entity.

```csharp
// Soft delete: never return soft-deleted rows unless explicitly asked.
builder.HasQueryFilter(c => !c.IsDeleted);

// Multi-tenancy: only return rows for the current tenant.
builder.HasQueryFilter(c => c.TenantId == _currentTenant.Id);
```

Every query against the entity automatically includes the filter, with no per-query code. It is the model-world answer to the same problem ECS solves with a `SoftDeleted` component and a filter system — less elegant in some ways, but built into the framework and requiring no discipline at the call site.

The caveat: query filters are *easy to forget you have*, and the day you genuinely need a soft-deleted row, you reach for `IgnoreQueryFilters()` and hope you remembered why.

## Keeping the domain unaware

The thread running through this chapter, and the reason the configuration-class pattern matters beyond mere tidiness: *the domain model should not know it is persisted*.

A `Customer` with `[Table]`, `[Column]`, `[Key]`, and `[MaxLength]` attributes is a domain model that has been told, in some detail, about the database it lives in. A `Customer` that is a clean domain class, configured entirely from `CustomerConfiguration`, does not know and does not care. You could swap the persistence layer — to a different database, to a document store, to an in-memory test double — and the domain class would not change a line.

This separation is not free; it costs you the configuration classes. It is, in my experience, one of the highest-return small disciplines available — the kind of thing that, eighteen months in, is quietly responsible for the codebase still being pleasant to work in.

## Performance, in passing

Configuration mostly happens once, at model-building time, and is cached for the application's lifetime, so its runtime cost is negligible. Two small notes nonetheless.

Value converters have a per-materialisation cost — each row read converts each converted property. For a `Guid`-wrapping value object this is unmeasurable; for a converter that does real work (encryption, compression, JSON serialisation) on a hot read path, it is worth knowing about.

Global query filters add a predicate to every query against the entity. The predicate is usually indexed (a `TenantId` column, an `IsDeleted` flag) and the cost is small — but an *unindexed* query filter on a large table is a full scan applied to every single query, which is the kind of thing that does not show up until the table is large and then shows up everywhere at once.

> *A query filter on an unindexed column is rather like a bouncer who checks every guest's identification by telephoning their mother — scrupulous, well-intentioned, and singlehandedly responsible for the queue now stretching around the block.*

## The smells

- Data annotations (`[Column]`, `[Key]`) on a domain entity — persistence concerns have leaked onto the domain.
- An `OnModelCreating` method longer than about fifty lines — it wants splitting into configuration classes.
- A value object stored as a string with manual `.ToString()` / `.Parse()` at every call site — it wanted a value converter.
- A soft-delete check (`.Where(x => !x.IsDeleted)`) repeated in every query — it wanted a global query filter.
- A configuration class that configures more than one entity type — one file, one entity.
- The internal `Id` appearing in a controller signature — the model-binding hookup is not wired up.

## Recap

- Configure entities in `IEntityTypeConfiguration<T>` classes — one per entity — not via annotations and not in a monolithic `OnModelCreating`.
- `ApplyConfigurationsFromAssembly` wires them up in one line; the explicit-list alternative is also fine.
- This is where the `PublicId` model-binding hookup, the value converters, and the query filters all live.
- The pattern keeps the domain model unaware that it is ever persisted, which is the whole point.
- Watch for unindexed query filters — they apply to every query and a missing index is felt everywhere.

## Onwards

The next chapter is the one where Part III and Part IV finally hold hands. We have spent Part IV on relational data, queries, and configuration; Part III argued for an architecture in which the natural unit is a component rather than a row. The obvious question — *how do you persist an ECS world to a relational database?* — has been waiting since [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md). It is time to answer it.
