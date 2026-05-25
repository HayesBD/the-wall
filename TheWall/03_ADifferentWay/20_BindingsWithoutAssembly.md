# Chapter 20 — Bindings Without Reflection-by-Assembly

There is a kind of code I have inherited several times in my career that I have come to recognise on sight. It always lives in `Program.cs` or `Startup.cs`. It always involves three or four nested calls that look something like:

```csharp
services.Scan(scan => scan
    .FromAssembliesOf(typeof(IService), typeof(IRepository))
    .AddClasses(c => c.AssignableTo(typeof(IService)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

The intention is admirable. The intention is *I do not want to write three hundred registration lines for three hundred service classes; I want the container to find them for me*. The result is admirable for the first few months. The result, by year three, is a startup process that takes thirty seconds, a registration trail no debugger can follow, and a particular flavour of bug — *"this implementation should have been picked up but wasn't, and we don't know why"* — that consumes the better part of a senior engineer's afternoon every time it appears.

I want to spend a chapter on the alternative shape that ECS quietly enforces, because the operational properties of *not* relying on assembly scanning are some of the most pleasant and least talked-about benefits of the pattern. The chapter is a small one, but the cumulative effect over the life of a project is large.

## The problem assembly scanning was meant to solve

To be fair to the scanning approach, the underlying problem is real. A mature business codebase with a hundred entity types, three hundred services, fifty repositories, and a few hundred view models genuinely is too many registrations to maintain by hand. The temptation to say *"find them all by convention"* is reasonable. The cost of giving in to it is what this chapter is about.

The cost has three parts.

**Startup time.** Every assembly scan walks the assembly's metadata, reflects on every type, checks every type against every convention, and registers the matches. For a small project this is milliseconds; for a large one it is *seconds*, sometimes tens of seconds. In a deploy-frequently shop, this is a chunk of time the team pays for, every time, on every restart.

**Opacity.** When `IFooService` is requested and resolves to `FooService`, the registration responsible for that mapping is *not visible in the source code*. It was discovered at runtime by a convention. New starters cannot find it. Tools cannot find it. The IDE's *"find references"* does not find it. The implementation appears, as if by magic, when the application runs.

**Mystery failures.** The flip side of opacity. When `IBarService` resolves to *the wrong implementation*, or to nothing at all, the diagnostic process is: *which convention should have caught it? Why didn't it? Was the assembly loaded? Was the type filter too narrow? Did someone change the namespace?* Each of these takes time. None of them is easily testable.

The fourth cost, which I will mention separately because it is structural rather than operational, is *that convention-based registration encourages convention-based code*. *"Every class ending in `Service` is a service"* tilts a codebase towards classes that earn the name `Service` only loosely, towards the `Manager`/`Helper`/`Util` suffixes we catalogued in [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md), and towards a general fuzziness about what each class is actually *for*.

## How ECS sidesteps the problem

The ECS pattern, almost by accident, makes the assembly-scanning approach unnecessary. The reason is that the things requiring registration in an ECS world — *components* and *systems* — are registered through mechanisms that the language already supports cheaply, without reflection.

**Components.** As we saw in [Chapter 19](./19_Registries.md), each component type gets a unique runtime ID via a static generic class:

```csharp
public static class ComponentTypeId<T>
{
    public static readonly int Value = ComponentRegistry.Register(typeof(T));
}
```

The first time any code accesses `ComponentTypeId<Position>.Value`, the static initialiser fires and registers `Position`. No scanning. No reflection beyond the one `typeof(T)`. No conventions. Components self-register on demand, in dependency order, automatically. The total cost is essentially zero — a single hash-map insert per component type, exactly once per process.

**Systems.** Systems are registered explicitly, in one place:

```csharp
public static class SystemRegistration
{
    public static void RegisterAll(World world, IServiceProvider services)
    {
        world.AddSystem(new MovementSystem());
        world.AddSystem(new LoyaltyPromotionSystem(services.GetRequiredService<IEventBus>()));
        world.AddSystem(new AuditSystem(services.GetRequiredService<IClock>()));
        world.AddSystem(new SoftDeleteFilterSystem());
        world.AddSystem(new TenancyFilterSystem(services.GetRequiredService<ITenantContext>()));
        // ...
    }
}
```

That looks like a lot of lines. It is, by design, a list. The list is *the entire inventory of what runs in this application*. A new starter can read it. A code reviewer can read it. The IDE's *"go to definition"* works on every line. The order of registration is explicit. The dependencies of each system are visible in its constructor call.

For a system that has fifty handlers, this is fifty lines. For one with three hundred, it is three hundred. The trade-off is real — you write the registration code by hand — but in exchange you get a startup that takes milliseconds, a wiring that is fully self-documenting, and a debugger that can step through every connection.

## Source generators close the gap

Modern C# offers a tidy compromise: a source generator can emit the explicit registration code for you, at compile time, based on attributes you put on your systems.

```csharp
[RegisterSystem]
public class MovementSystem { /* ... */ }

[RegisterSystem]
public class LoyaltyPromotionSystem(IEventBus events) { /* ... */ }
```

The generator scans your project *at compile time* — not at startup — and emits something equivalent to the explicit `RegisterAll` above. You get the brevity of attribute-based registration. The runtime gets the speed and clarity of explicit registration. The IDE gets the full referenceability of generated code. Everybody wins.

This is a relatively recent capability — source generators landed in C# 9, and the toolchain has matured enough to lean on heavily only in the last few years. Older ECS frameworks predate it and use reflection, the way DI containers do. New ones lean on it heavily. The cost difference at scale is, depending on the application size, between *significant* and *transformative*.

## A small comparison

To make the contrast concrete, here is the same registration shape in three styles.

**Assembly scanning (DI):**

```csharp
services.Scan(scan => scan
    .FromAssembliesOf(typeof(ISystem))
    .AddClasses(c => c.AssignableTo<ISystem>())
    .AsImplementedInterfaces()
    .WithSingletonLifetime());
```

Four lines. Slow startup. Opaque. Hard to debug.

**Explicit registration (ECS, hand-written):**

```csharp
world.AddSystem(new MovementSystem());
world.AddSystem(new LoyaltyPromotionSystem(eventBus));
world.AddSystem(new AuditSystem(clock));
// ... fifty more lines
```

Many lines. Fast startup. Self-documenting. Easy to debug.

**Attribute + source generator (ECS, modern):**

```csharp
[RegisterSystem] public class MovementSystem { /* ... */ }
[RegisterSystem] public class LoyaltyPromotionSystem(IEventBus events) { /* ... */ }
[RegisterSystem] public class AuditSystem(IClock clock) { /* ... */ }
```

One attribute per system. Compile-time generation. Fast startup. Self-documenting. Easy to debug.

The third option is the natural place for a new C# ECS project to start. The middle option is the right reach for teams who prefer to keep the wiring explicit and visible. Both are *categorically* nicer to live with than the first.

## What this is not

I want to be careful here. None of the above is an argument against dependency injection. DI is a fine and useful tool; the *container* is the right mechanism for constructing services with non-trivial dependency graphs; the application is well-served by knowing how to use one.

The argument is narrower: *assembly scanning as the default registration mechanism is a false economy that compounds with codebase age*. Whether you adopt ECS or not, the same argument applies — explicit registration scales better, both as a development experience and as a runtime cost, than convention-based discovery.

In an ECS architecture, the argument is reinforced because the entities requiring registration are so well-behaved (components self-register through type IDs; systems are an enumerable list) that there is no temptation to scan an assembly in the first place. The pressure that produces scanning-by-default in DI-heavy codebases simply does not arise.

## Performance, in passing

The numbers depend on the codebase, so take what follows as representative rather than universal.

A medium-sized .NET application with a few hundred services and a dozen scanning conventions typically spends *one to three seconds* on container construction at startup. The same application with explicit registration typically spends *fifty to two hundred milliseconds*. The same application using compile-time source generation typically spends *under twenty milliseconds*.

For a long-running web server, this is one-time and arguably immaterial. For a serverless function with cold-start latency in the user-visible request path, it is the difference between *acceptable* and *unacceptable*. For a development feedback loop that involves frequent restarts — TDD, integration test runs, hot reload — it is the difference between *pleasant* and *exhausting*.

The performance cost of opacity is harder to quantify but, in my experience, larger. Hours per week, per developer, lost to *"where does this resolve from?"* and *"why didn't this register?"* questions are not measured but are real.

> *Trusting an assembly scanner to find the right registrations in a five-year-old codebase is rather like trusting a labrador to find your missing socks: there will be considerable enthusiasm, a great deal of activity, and on the whole rather less success than you had hoped for.*

## The smells

- A `Startup.cs` whose registration block has more conditionals than lines of actual `AddSingleton`.
- A library called `Scrutor` (or similar) in the dependency list and a comment above its usage that says *"do not touch — it took us a week to get right."*
- A new starter's first ticket consists almost entirely of *"why doesn't my new service get picked up?"*
- A startup time that crosses one second on a developer machine and nobody can quite say why.
- A registration that resolves correctly in production and incorrectly in tests, or vice versa.

## Recap

- Assembly scanning in DI containers is a reasonable-looking shortcut whose costs compound with codebase age.
- The three main costs are startup time, opacity, and mystery failures.
- ECS makes scanning unnecessary by design — components self-register through static type IDs; systems are explicitly registered or attribute-marked.
- Source generators close the convenience gap, giving you attribute-based brevity with compile-time generation.
- The argument is not against DI. The argument is against scanning-as-default.

## Onwards

The next chapter is the ECS equivalent of [Chapter 14](../02_TheModelQuestion/14_ModelSmells.md) — the bestiary of things that go wrong in an ECS codebase. Bloated components that drift back towards being models. Systems that secretly hold state. Registries that become god objects. The seductive over-componentisation that turns simple changes into a hunt across twelve files. Yes, ECS has its own catalogue of trouble; honesty requires us to spend a chapter on it before we put the two approaches side by side in [Chapter 22](./22_ModelsVsECS.md).
