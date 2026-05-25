# Chapter 6 — Generics, and a Word on the Competition

There are perhaps three things in C# I would, without hesitation, hand to a designer of a new language and say *"do it this way."* Generics is one of them.

This was not always going to happen. The first version of C# did not have generics. The first version of Java did not have generics either. We had `ArrayList`, which held `object`s, and we had a great many boilerplate wrapper classes named `StringList` and `OrderList` and `IntegerList`, written by exasperated engineers trying to claw type safety back from a runtime that did not understand the question. The era was short and not greatly missed.

When generics arrived — for C# in 2005, for Java a year before — the experience was transformative in two distinct ways. The boilerplate evaporated overnight. And, less obviously but rather more importantly, a great deal of accidental boxing evaporated with it.

## What a generic actually is

A generic is a type or method with one or more type *parameters* — placeholders for real types, filled in at the call site, checked by the compiler.

A generic type:

```csharp
public class Box<T>
{
    public T Contents { get; }
    public Box(T contents) => Contents = contents;
}

var intBox    = new Box<int>(42);
var stringBox = new Box<string>("hello");
```

A generic method:

```csharp
public static T FirstOrThrow<T>(IEnumerable<T> source) =>
    source.FirstOrDefault() ?? throw new InvalidOperationException("Empty.");

int  first = FirstOrThrow(new[] { 1, 2, 3 });
User user  = FirstOrThrow(users);
```

One honest subtlety, very much in the spirit of this book: that `?? throw` does the right thing for *reference* types, but for a *value* type `T`, `FirstOrDefault()` returns `default(T)` — `0` for an `int`, not null — so on an *empty* sequence it would return the default rather than throwing. If that edge case matters, constrain `where T : class`, or reach for `.First()`, which already throws on empty. The compiler's nullable analysis is, here, only as honest as the types allow.

In both cases the *real* type — `int`, `string`, `User` — is settled at compile time. The method body operates on a `T` that the compiler treats consistently, the call site receives a `T` that the compiler treats correctly, and nothing is boxed, nothing is cast, nothing is checked at runtime that could be checked earlier.

That last sentence is worth pausing on. It is the difference between a feature that is *type-safe* and a feature that is *type-erased*. C# generics are the former: each `List<int>` and `List<string>` is a genuinely distinct type at runtime, with no `object`-shaped pretence in between. Other languages we will get to in a moment are the latter.

## Constraints — telling the compiler what `T` can do

Without constraints, the only operations the compiler will let you perform on a `T` are the ones available on `object` — `Equals`, `GetHashCode`, `ToString`. The moment you want to do more, you constrain.

```csharp
public T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) >= 0 ? a : b;

public T Make<T>() where T : new() => new T();

public void Process<T>(T item) where T : class, IDisposable
{
    using (item) { /* ... */ }
}
```

The full set of constraints — `class`, `struct`, `notnull`, `unmanaged`, `new()`, base classes, interfaces, and the more recent `where T : IFoo<T>` for static abstract members — is one of the most quietly powerful corners of the language. It lets you write generic code that is, in practice, almost as expressive as code written for a specific type, without losing the genericity.

## Variance — a short word, used sparingly

You will encounter, on certain interfaces, the keywords `out` and `in` on a type parameter:

```csharp
public interface IEnumerable<out T> { /* ... */ }
public interface IComparer<in T>    { /* ... */ }
```

`out` makes the type parameter **covariant** — an `IEnumerable<Dog>` may be assigned to an `IEnumerable<Animal>`, because every dog is an animal and the interface only ever produces `T`s. `in` makes the type parameter **contravariant** — an `IComparer<Animal>` may be assigned to an `IComparer<Dog>`, because a thing that can compare any animal can certainly compare two dogs.

This sounds abstract. In practice you mostly *use* variance — when you assign a `List<Dog>` to an `IEnumerable<Animal>` and it just works — rather than *declare* it. Declaring variance is for the library authors among us.

## Where generics earn their keep

A non-exhaustive tour of the patterns generics make pleasant:

- **Collections.** `List<T>`, `Dictionary<TKey, TValue>`, `HashSet<T>` and friends. The end of `ArrayList` and the four thousand wrapper classes.
- **Repositories.** `IRepository<TEntity, TKey>` as a single shape that serves every entity in your domain.
- **Result types.** `Result<TValue, TError>` for honest failure modes. (See [Chapter 5](./05_DiscriminatedUnions.md) for what to put inside one.)
- **Mediators and handlers.** `IRequestHandler<TRequest, TResponse>` patterns — the bones of MediatR and similar libraries.
- **Builders and fluent APIs.** Returning `TSelf` from each step so the chain stays well-typed.

In each case the generic version is shorter, safer, and faster than the non-generic equivalent.

## A polite word about the competition

I want to be careful here, because nothing is duller than a language partisan with a grievance, and the people who write Python and JavaScript are, on the whole, not the problem. The languages, however, have a specific and persistent gap in this exact corner.

Python has type hints. They are excellent for documentation, decent for static analysis, and entirely *advisory* at runtime. `def first(items: list[int]) -> int` is a polite request that the IDE will respect and the interpreter will ignore. There is no `List<int>` at runtime — there is only `list`, holding whatever it happens to hold, and you find out the moment you ask it. The four pillars of object orientation are observed; one or two of them are observed virtually.

JavaScript has no generics at all. TypeScript, sitting on top, has very serviceable generics — but they are compile-time only, erased before the runtime sees them, and at the moment of truth what you have is a plain JavaScript object with no recollection of its type parameters. TypeScript generics are valuable; let no one tell you otherwise. They are not, however, what we have in C#.

The practical consequence, in both languages, is more defensive code, more runtime checks, more dictionaries-of-anything-keyed-on-strings, more bugs that the compiler could have caught and did not. Both languages have their place — Python in particular is a rather wonderful tool for a great many jobs — but neither will, at three in the morning, refuse to compile a function that puts a `Customer` into a `List<Order>`. C# will, and you will sleep better for it.

> *Trying to scale a complex application on optional runtime typing is rather like setting out across the Atlantic in a perfectly nice rowing boat: the boat is fine; the journey is the wrong shape for it.*

## Performance, in passing

The single most underappreciated benefit of generics is: no boxing. A `List<int>` stores ints, contiguously, as ints. An `ArrayList` holding ints stores each one as a boxed `object` — a heap allocation per element, a layer of indirection on every read. For a list of a million ints, that is a million heap allocations the generic version simply does not make.

Constraint-driven generics with structs are even better — the runtime *monomorphises* generic methods for value-type parameters, generating a specialised version per concrete type. There is no dispatch, no boxing, no virtual call. The cost of `Max<int>(a, b)` is the cost of `int`'s `CompareTo`, and nothing more.

## The smells

- A method taking `object` and casting inside — you wanted a generic method.
- A method taking `dynamic` for the same reason — same answer, and you have given up the compiler's help besides.
- A `Dictionary<string, object>` that everywhere needs `(SomeType)dict["key"]` to use — you wanted a generic dictionary, or a record, or a class.
- A generic type with no constraints whose body still casts `T` to a specific type — you have not actually written a generic method; you have written a confused specific one.
- A `List<int>` round-tripped through `ArrayList` for any reason. There is no good reason.

## Recap

- Generics give you type-safe reuse with no runtime cost.
- Constraints make generic code as expressive as specific code without losing genericity.
- Variance is mostly for library authors; the rest of us just consume it.
- The lack of generics in dynamically typed languages is paid for in defensive code, runtime checks, and three-in-the-morning bugs.
- A `List<T>` does not box. An `ArrayList` does. This matters more than people realise.

## Onwards

The next chapter steps slightly sideways, away from *what types are* and into *how objects come into being*. C# stamps instances from blueprints; JavaScript bolts them onto prototypes; Python does something altogether more curious. Understanding the difference is a quiet superpower, even if you never write a line of either of the others.
