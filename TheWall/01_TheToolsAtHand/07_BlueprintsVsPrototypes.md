# Chapter 7 — Blueprints vs Prototypes: How Objects Get Made

Every language that calls itself object-oriented has to answer one fundamental question, and the answers it gives shape almost everything about what writing in that language *feels* like. The question is: when you create an object, where does its *shape* come from?

There are three serious answers to this question, three philosophies of object construction, and each of the three major mainstream languages — C#, JavaScript, and Python — has chosen a different one. I think it is worth spending a chapter here, because almost every C# developer I have met has spent some time writing one of the other two, and almost none of them has noticed how deeply the choice colours their thinking when they switch back.

## The blueprint

C# is a *blueprint* language. The shape of an object is declared, in advance, by its class. The compiler stamps every instance from that declaration, and every instance of `Customer` has exactly the same properties, methods, and fields as every other instance of `Customer`. You cannot, after the fact, bolt a `LoyaltyTier` field onto one particular customer because today happens to be Tuesday.

```csharp
public class Customer
{
    public string Name { get; init; }
    public DateOnly JoinedOn { get; init; }
}

var c = new Customer { Name = "Alice", JoinedOn = new(2025, 1, 1) };
// c.LoyaltyTier = "Gold"; // compile error — there is no such property
```

This is, in the world of programming language design, a relatively strict choice. It is also why the C# compiler can give you the help it gives you — IntelliSense, refactor-rename across a million-line codebase, the confidence that a thing called `Customer` has the shape you think it has — because the *shape itself* is a first-class compile-time artefact.

Inheritance, in a blueprint language, is similarly upfront: declared with a colon, fixed at compile time, walked by the runtime through a vtable you cannot edit. *"`Spaniel` inherits from `Dog` inherits from `Animal`"* is a sentence that means the same thing to every spaniel for the rest of the program.

## The prototype

JavaScript is a *prototype* language. There are no classes in the C# sense; the `class` keyword that arrived in ES2015 is, underneath, sugar over the older prototype machinery. The actual model is this: every object has a hidden link — historically `__proto__`, properly `[[Prototype]]` — to another object, and property lookup walks that link until it finds the property or runs out of objects to walk.

```javascript
const customer = { name: "Alice", joinedOn: "2025-01-01" };
customer.loyaltyTier = "Gold";   // perfectly fine
delete customer.joinedOn;        // also fine
customer.greet = function() { return `Hi, ${this.name}`; }; // please do not
```

Any object can have any property at any time. The "shape" of a customer is whatever the customer happens to be carrying when you ask. There is no contract. There is no compile-time check. There is, in exchange, a great deal of flexibility — and a great deal of mischief.

This is the kind of choice that pays for itself when you are writing fifty lines of JavaScript to glue a button to a callback. It bills you, with interest, when you are writing fifty thousand lines.

> *A JavaScript object is less a thing-with-a-shape than a thing-with-a-current-mood; it has whatever properties it feels like having today, and it cordially invites you to keep up.*

## Python — the middle ground that is not quite the middle

Python looks, at first glance, more like C# than JavaScript. There are `class` declarations. There is `self`. There is inheritance with a fixed list of bases. The four pillars of object orientation — encapsulation, abstraction, inheritance, polymorphism — are all present and correct.

It is the *enforcement* that is curious.

In Python, every instance is really a dictionary. The `__dict__` attribute on any object will show you the actual mapping of names to values. Class methods live in the class's `__dict__`; instance attributes live in the instance's `__dict__`. Attribute access walks the chain: instance first, then class, then base classes via the Method Resolution Order.

```python
class Customer:
    def __init__(self, name):
        self.name = name

c = Customer("Alice")
c.loyalty_tier = "Gold"   # perfectly fine — bolted on at runtime
del c.name                # also fine
print(c.__dict__)         # {'loyalty_tier': 'Gold'}
```

Encapsulation is, famously, *by convention*. A leading underscore means *please don't touch this*. Two leading underscores invoke name mangling, which makes touching it inconvenient but not actually impossible. None of this is enforced by the runtime in the way C# enforces `private`. It is a polite request.

Polymorphism is by duck typing — if it walks like a duck, the runtime is satisfied, and the static type system, such as it is via type hints, will note its concerns and then stand aside. Inheritance is real, but you can also reassign `__class__` on an instance to change its class at runtime if you really want to, and the language will not stop you.

The four pillars are observed. One or two of them are observed virtually.

## Why this matters to a C# developer

You may, reading the above, be thinking: I write C#, I don't need to worry about any of this. I have some sympathy. Here is why I think the chapter earns its place.

**You will work with people who think prototypically.** A team-mate coming from a JavaScript background will, by instinct, reach for the bag-of-properties approach. They will favour the `Dictionary<string, object>` over the typed class, will design APIs that accept *anything*, will tell you that *"we'll add a field later if we need it."* This is not stupidity; it is muscle memory from a language where that approach was the *right* one. The conversation goes better when you can name the difference rather than just feeling vaguely cross about it.

**You will read JSON.** All of it, every day, for the rest of your career. JSON is the wire format of a prototype-shaped world — keys and values, no schema, no contract, whatever the producer felt like sending. The C# instinct is to parse it into typed records at the boundary and never let the prototype-shaped soup into the application itself. (This is the right instinct.)

**You will be tempted by `dynamic` and `ExpandoObject`.** Both exist in C# precisely to interoperate with prototype-shaped systems. Both are useful in their narrow context — COM interop, IronPython hosting, the occasional `RouteValueDictionary` — and disastrous when used as a general escape hatch from the blueprint model. If you find a project that uses `dynamic` in its core domain, run.

**You will write code that *should* be blueprint and *is* prototype-shaped.** A class with thirty nullable properties, half of which are populated under different conditions, is a JavaScript object pretending to be a C# class. You wanted either a discriminated union (see [Chapter 5](./05_DiscriminatedUnions.md)) or several distinct types. You got a bag.

## Performance, in passing

Blueprint languages have a quiet performance advantage that is easy to undersell. Because the shape is known at compile time, the runtime can lay an object out as a contiguous block of fields at fixed offsets, dispatch methods through a vtable, and skip every form of dynamic lookup. Property access is a pointer-plus-offset. Method calls resolve in a single indirection.

Prototype languages cannot do this without help. Every property access is, in principle, a chain walk; every method call is a name lookup. Modern JavaScript engines work miracles to compile this down — hidden classes, inline caches, all the JIT cleverness one could wish for — but the *baseline* model is slow, and falling off the JIT's fast path is one line of code away.

Python is more honest about the cost. Every attribute access is a dictionary lookup, and CPython has never pretended otherwise. NumPy, Pandas, and the rest of the scientific stack exist precisely because, for hot numerical work, the dictionary-of-attributes model is hopeless and the answer is to drop into C and treat Python as the conductor rather than the orchestra.

## The smells (mostly C# trying to act like one of the others)

- A C# class containing a `Dictionary<string, object>` for *"extension properties"* — you wanted JavaScript and you should not have.
- An API that accepts `dynamic` parameters for anything except COM interop — you have surrendered the compiler's help on purpose.
- A method that takes a `JObject` and pulls fields out of it by string keys — you stopped at the parse boundary too early; finish the typed conversion.
- A class with thirty `string?` properties named like `ExtraInfo1`, `ExtraInfo2` — you have built a prototype object the slow way.
- The phrase *"we can just bolt that on later"* describing a domain entity — that is the prototype mindset describing what should be a blueprint decision.

## Recap

- C# is a blueprint language: shape declared, compiler-enforced, instances stamped from the class.
- JavaScript is a prototype language: shape is fluid, properties bolted on at will, no contract.
- Python sits between them in form but closer to JavaScript in enforcement: classes that look like blueprints, instances that behave like dictionaries, pillars observed by polite convention.
- The blueprint model is the source of much of what makes C# pleasant to scale; do not casually invite the other models into the core domain.

## Onwards

The next chapter takes the most misunderstood feature of modern C# — `async`/`await` — and explains what the compiler is actually doing on your behalf. After that, threading, parallelism, and the small number of situations in which you actually need to think about a thread by name.
