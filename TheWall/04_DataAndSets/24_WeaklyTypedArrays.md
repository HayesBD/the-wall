# Chapter 24 — A Short Polemic on Weakly Typed Arrays

This chapter is short by design. I promised it several chapters ago, when we were discussing generics ([Chapter 6](../01_TheToolsAtHand/06_Generics.md)) and again when we were discussing collections in the chapter just past. It is the part of the series where I allow myself, for a few hundred words, to be a small amount partisan about what C# gets right that some other widely-used languages do not.

The complaint is narrow. The complaint is about the word *array*, and what that word means in C#, JavaScript, and Python — three languages that all claim to have *arrays* and three languages whose claims, on close inspection, are doing very different work.

I will keep the chapter brief. Four observations, then a small concession, then we move on.

## Observation one — element types

In C#, the type of `int[]` is, with delightful unambiguity, *an array of ints*. You cannot put a string in it. You cannot put a Customer in it. The compiler will refuse, the runtime will refuse, every tool in the ecosystem will refuse, and you would have to go to considerable effort to make any of them change their mind.

In JavaScript, `[1, "hello", true, null, {name: "Alice"}, undefined]` is a perfectly legal array. The runtime is content. The interpreter is content. The tooling, in 2026, will *warn* you if you have configured it to do so, but the language itself is entirely happy. The same array, two minutes later, might be `[1, "hello", true, null, {name: "Alice"}, undefined, function(){}, new Date()]`, with no notification to anyone.

In Python, `[1, "hello", True, None, {"name": "Alice"}]` is, similarly, a perfectly legal list. Type hints can document a more constrained intention; the interpreter will not enforce it.

I will not labour the point. *A type that promises nothing about what it contains is a type that has surrendered most of what types are for.*

## Observation two — performance opacity

A C# `int[]` is a contiguous block of memory containing four-byte integers laid out one after the other. The CPU knows this. The CPU's prefetcher knows this. Walking the array is as fast as walking a contiguous block of memory ever gets.

A JavaScript array, depending on the engine and the contents, may be a contiguous block, or a sparse-indexed object, or a hash map keyed on numeric strings, or some adaptive mixture of all three. Modern V8 will try hard to keep it fast; if you put anything *odd* into it — `delete array[5]`, or `array[1000000] = "hello"` — the engine quietly downgrades the storage and you no longer have what you thought you had. Most JavaScript developers I have asked could not tell me, without checking, what their hottest array's current backing storage actually is.

A Python list is always a list of pointers to PyObjects on the heap. Each access is a pointer dereference. Each element costs a full PyObject's worth of metadata. NumPy exists, in some real sense, *because* this is the case.

*A collection whose memory layout is unknowable is a collection whose performance is unknowable.*

## Observation three — compile-time guarantees

C# offers, at the moment a function declares `int[] Foo(int[] input)`, a guarantee that every call site will pass an `int[]` or face the compiler. The body of the function can rely, without checking, on every element being an integer. The IDE can offer accurate completion. The refactor tools work.

JavaScript and Python offer none of this from the language proper. TypeScript and Python's type hints offer some of it, advisorily, at compile time, with the runtime making no commitment of its own. The function body may, in principle, assume the type hint was honoured; the function body that ships to production usually adds a defensive check or two anyway, because the function body's author has been burned before.

The compile-time guarantees of strong typing show up as *the absence of code that would otherwise need to exist*. Every defensive check that a strongly-typed signature spares you is a small win. Across a codebase of any size, it adds up.

## Observation four — the cheerful acceptance

The most persuasive case for the weakly-typed approach is *flexibility*. Dynamic languages permit a kind of casual, quick, exploratory programming that is genuinely useful for some workloads — scripting, scientific exploration, glue code, prototypes. The bath-water of strict typing does, at the lower end of program complexity, throw out some of this baby.

The case turns, however, on the complexity of the program. For a thirty-line script, the casual approach is correct. For a thirty-thousand-line application, the casual approach has been paid for in defensive checks, runtime type errors, refactoring pain, and the kind of bug that lives quietly for six months before announcing itself in front of a customer.

> *Defending dynamic typing for serious software because it is "more flexible" is rather like defending the absence of a seat belt because it lets you lean further forward — there is a sense in which the claim is true, and there is a longer sense in which one is rather glad of the constraint.*

## The concession

I do not want to be unfair, and the chapter would be incomplete without acknowledging this directly: *the dynamic-typing approach to collections is the right answer for some kinds of work, and the people who choose it are usually choosing it for defensible reasons*.

Notebook-driven data exploration, where the shapes of things are unknown until you have looked. Quick scripts to munge log files. Glue code at system boundaries where the inputs are genuinely heterogeneous and the cost of typing them all out exceeds the cost of being careful at runtime. For all of these, the dynamic-typing approach is at worst comparable to the strongly-typed one and often genuinely nicer.

The case for strong typing is a case at *scale*. As applications grow, the relative value of compile-time guarantees grows with them. The reason mainstream business software is mostly written in strongly-typed languages — and the reason it has been for the entire history of the field — is that, by the time a system is two years old and serving real customers, the cost-benefit has tipped decisively in that direction.

## Recap

- *"Array"* means very different things in C#, JavaScript, and Python.
- C# arrays are type-safe, performance-predictable, and compile-time-checked.
- The dynamic alternative is genuinely useful for small scripts and exploratory work, and a poor fit for large applications.
- The case for strong typing is a case at scale, not a case for every line of code.

## Onwards

The next chapter steps from the practical question of *which collection* to the mathematical question of *what collections actually are*. Set theory, explained for an audience of unsophisticated five-year-olds and a small minority of slightly impatient adults, and the lovely fact that most of LINQ turns out to be the same mathematical idea wearing different hats.
