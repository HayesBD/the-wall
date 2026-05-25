# Chapter 40 — Onwards

We have arrived, somewhat to my surprise and possibly to yours, at the end.

It began, as the foreword admitted, as a private attempt to work out what I actually thought — to drag a set of foggy convictions into the daylight and see whether they had legs. Forty chapters later I am reasonably confident that most of them do, and rather better at explaining the few that turned out to be merely opinions. Whether the exercise has been as useful to read as it was to write is, of course, entirely your call to make, and not one I can make for you.

Before we part, a short accounting.

## The two quiet threads

There were two threads running beneath the surface of this book, and now that we are at the end I can acknowledge them.

The first I named openly: *the right way and the fast way have a stubborn habit of being the same way.* Across forty chapters, the recommendation that produced cleaner code produced — more often than not, and usually without anyone setting out to make it so — faster code as well. Lean models materialise more cheaply. Value objects as structs cost nothing. Projection beats fetching the world. The right collection dodges the Cartesian product. Stateless systems parallelise for free. This was not a coincidence I engineered for effect; it is, I have come to believe, a genuine property of the craft. Tidiness and performance are not opponents to be traded against one another. They are, with remarkable frequency, the same virtue seen from two angles.

The second thread I will leave where it is. You have read Parts II and III; you have seen the case study built twice and read the diff between the milestones. I made an argument, as fairly as I could manage, with both sides given their best showing and their honest failure modes, and I declined at every turn to tell you what to conclude from it. I am not going to break that discipline in the final chapter. You have the evidence. You have your own system, your own team, your own roadmap. The conclusion was always yours to draw, and I would rather you drew it than borrowed mine.

## What this book deliberately left out

A book that tried to cover everything would have covered nothing well, so a great deal was left on the cutting-room floor on purpose. In the interest of honesty, here is some of what is missing and where it might be found.

**Testing strategy.** I have leaned on testability as a *property* throughout — stateless systems are easy to test, pure functions need no mocks, the captive-dependency bug is caught by a fixture — but I have not written the chapter on *how to test well*: the shape of a good test suite, the unit/integration/end-to-end balance, the discipline of testing behaviour rather than implementation. It deserves a book of its own, and several good ones exist.

**Performance profiling in earnest.** The *"Performance, in passing"* sections made the case that good structure tends to be fast, but they were, as advertised, *in passing*. The serious craft — reading a flame graph, wielding BenchmarkDotNet, understanding the garbage collector's generations, finding the actual hot path rather than the one you assumed — is a deep discipline this book only gestured at.

**Distributed systems.** Everything here assumed, more or less, a single application talking to a single database. The moment you have several services, a message bus, eventual consistency, and the fallacies of distributed computing to contend with, a whole new category of concern opens up — and almost none of it is covered here.

**Security.** Authentication and authorisation got a paragraph in the middleware chapter; the public-id pattern closed off one enumeration vulnerability. But the OWASP top ten, secure-by-design, threat modelling, the handling of secrets — these are conspicuous by their absence, and their absence should not be read as their unimportance.

**The front end.** This was a book about the shape of code behind the API. What happens in the browser — the component models, the state management, the rendering — is an entire discipline this book did not touch.

Each of these is a thread I chose not to pull, so that the threads I did pull could be followed to their ends. A book is defined as much by what it declines to be as by what it is.

## Where to go next

The appendices are the natural next stop within these covers. [Appendix B](../appendices/B_ReadingList.md) is the honest reading list — the books, papers, and talks that shaped what is here, several of which go far deeper into the topics above than I have. [Appendix E](../appendices/E_RefactoringOrder.md) is the one to open if you are looking at an existing codebase and wondering where to start; it lays out the order in which to apply everything here without setting fire to the thing that is currently paying the bills.

And then, of course, there is the work itself. None of this lands by reading. It lands when you reach, in the actual code in front of you on an actual Tuesday afternoon, for the value object instead of the bare string, the guard clause instead of the nested pyramid, the result type instead of the thrown exception — and find, a few months later, that the reaching has become a habit, and the habit has quietly made the codebase a more pleasant place to spend your days.

## A short farewell

I said at the outset that this was not, I hoped, a sermon. I have tried to keep that promise — to tell you what I think and why, to make the case as fairly as I could, and to leave the conclusions to you. If I have occasionally raised my voice (the exceptions chapter springs to mind), I hope it was in proportion to the provocation.

If you have found something cleaner than what is here — a better way to do one of these things, an argument I got wrong, a conviction of mine that turns out to be merely an opinion after all — I meant it on the first page and I mean it on the last: I would genuinely like to hear about it. The heart full of curiosity I mentioned in the foreword is, if anything, fuller now than when we began.

Thank you for reading. It has been a genuine pleasure to have the company.

Now go and write something good.
