# Appendix B — A Reading List

No book springs from nothing, and this one owes more than it can fully repay to the works below. They are grouped by the part of this series they most influenced, with an honest line on what each one is and why it is here. Several go far deeper than I have into topics I treated only in passing; where this book and one of these disagree, the reader is invited to weigh the argument rather than the author.

## On models, design, and refactoring

- **Martin Fowler, *Refactoring: Improving the Design of Existing Code*.** The origin of the code-smell vocabulary that Parts II and VI lean on throughout. If you read one book from this list, and you have not read this one, read this one.
- **Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*.** The case for the rich domain model and the source of *bounded context*. The fair hearing the model approach gets in [Chapter 11](../02_TheModelQuestion/11_RichVsAnemic.md) is, in large part, Evans's case restated.
- **Martin Fowler, *Patterns of Enterprise Application Architecture*.** Where the term *anaemic domain model* was sharpened into a pejorative, and a reference for the persistence patterns underlying Part IV.
- **John Ousterhout, *A Philosophy of Software Design*.** Short, opinionated, and excellent on depth, complexity, and the cost of the wrong abstraction. A kindred spirit to this book's stance on speculative generality.
- **Steve McConnell, *Code Complete*.** The encyclopaedic elder statesman; the early-returns and naming arguments of Parts V and VI have roots here.

## On C# and .NET specifically

- **Jon Skeet, *C# in Depth*.** The definitive tour of how the language actually works underneath, including the generics and async machinery that Chapters 6 and 8 only sketched.
- **Bill Wagner, *Effective C#* and *More Effective C#*.** Item-by-item best practice; a natural companion to Part I.
- **Mark Seemann & Steven van Deursen, *Dependency Injection Principles, Practices, and Patterns* (Manning).** The book to read after [Chapter 31](../05_Architecture/31_DependencyInjection.md) if you want the full, rigorous treatment of DI done well — including a far deeper account of why composition roots and explicit registration beat the assembly scan. (The revised, expanded successor to Seemann's earlier *Dependency Injection in .NET*.)
- **Stephen Toub's writing on .NET performance and async** (the .NET team's engineering blog). The authoritative source on what `async`/`await`, `ValueTask`, and `Span<T>` actually cost; the ground truth beneath this book's *"Performance, in passing"* notes.
- **Andrew Lock, *ASP.NET Core in Action*, and his blog.** Excellent on the middleware pipeline of [Chapter 32](../05_Architecture/32_Middleware.md) and the framework internals around it.

## On data, sets, and querying

- **E. F. Codd, "A Relational Model of Data for Large Shared Data Banks" (1970).** The paper that translated set theory into the relational model and, by extension, into every query you write. The intellectual root of [Chapters 25](../04_DataAndSets/25_SetTheoryForFiveYearOlds.md) and [26](../04_DataAndSets/26_FiltersJoinsGroupings.md).
- **Paul R. Halmos, *Naive Set Theory*.** A famously gentle and short introduction to the mathematics underlying Part IV, for the reader who enjoyed the five-year-olds chapter and wants the grown-up version.
- **The EF Core documentation.** Genuinely excellent, and the reason [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md) declined to restate the API — it covers querying, change tracking, configuration, and JSON columns better than any third-party source.

## On ECS, data-oriented design, and the second half of the argument

- **Mike Acton, "Data-Oriented Design and C++" (CppCon 2014).** The talk that crystallised the data-oriented argument for a generation of engineers. Combative, clarifying, and the spiritual ancestor of much of Part III's performance reasoning.
- **Richard Fabian, *Data-Oriented Design*.** The book-length treatment of the same ideas; where the cache-coherence and structure-of-arrays arguments of [Chapter 19](../03_ADifferentWay/19_Registries.md) are developed in full.
- **The source code and documentation of *EnTT* (C++), *flecs* (C/C++), Unity *DOTS*, and *Bevy* (Rust).** The living, production-grade ECS frameworks referenced throughout Part III. Reading a real registry implementation teaches more about [Chapter 19](../03_ADifferentWay/19_Registries.md) than any prose can.
- **Sander Mertens (author of *flecs*) and the broader "ECS beyond games" discourse** — the *ecs-faq* and assorted essays arguing ECS as a general alternative to OO. Useful context: the idea that ECS could replace OO models outside games is not this book's invention; what is new here is the depth treatment for business C# and the way the case is measured ([Chapter 39](../07_TogetherNow/39_CaseStudy.md)).
- **Ivan Sutherland, *Sketchpad: A Man-Machine Graphical Communication System* (PhD thesis, MIT, 1963).** The historical precursor invoked in [Chapter 16](../03_ADifferentWay/16_ECSFirstLook.md): master/instance composition and a constraint solver, sixty years ahead of mainstream CAD. Not ECS, but the same long arc away from rigid structure. (Republished as Cambridge Computer Laboratory Technical Report UCAM-CL-TR-574.)

## On working with existing systems

- **Michael Feathers, *Working Effectively with Legacy Code*.** The definitive treatment of the characterisation tests and seams that [Appendix E](./E_RefactoringOrder.md) depends on. The book to read alongside that appendix.
- **Martin Fowler, "StranglerFigApplication" (martinfowler.com).** The short article that named the migration strategy at the heart of Appendix E.

## On software evolution and decay — the canon behind "the wall"

[Chapter 15](../02_TheModelQuestion/15_TheWall.md) builds on a well-established body of work. The book's contribution is a structural diagnosis and a metric, not the observation that software decays — which is documented here:

- **Meir M. Lehman, "Programs, Life Cycles, and Laws of Software Evolution" (*Proceedings of the IEEE*, 1980).** The foundational statement of the Laws of Software Evolution: a system in use must change, and its complexity rises unless work is done to reduce it.
- **Brian Foote & Joseph Yoder, "Big Ball of Mud" (PLoP '97, 1997).** The pattern paper that named the most-deployed architecture of all: the haphazard, sprawling system that accretes over time. The vivid ancestor of "the wall."
- **Ward Cunningham, the *technical debt* metaphor (OOPSLA '92 experience report).** The framing that the shortcuts you take today accrue interest you pay tomorrow.

## A closing note on the list

This is a list of influences, not an appeal to authority. Every book here has been argued with as much as it has been agreed with, and a reader who comes away from this series disagreeing with me is in excellent company — several of the authors above would disagree with me too, and with each other, and the field is healthier for it. Read widely, argue generously, and trust the code in front of you over the book on the shelf when the two conflict.
