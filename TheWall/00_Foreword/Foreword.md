# Foreword

This series began as a private exercise in trying to work out what I actually thought.

Anyone who has spent a few years writing software for a living knows the feeling: a strong intuition that *this is the right way* and *that is the wrong way*, formed from a great many late evenings and a small number of disasters — and then a meeting in which a perfectly intelligent colleague proposes the wrong way, and you find, to your horror, that you cannot quite explain why. You know. They know you know. They are unmoved. The conversation moves on. The wrong way ships.

So I started writing things down. Not to be right at anyone, but because the act of writing has a way of dragging foggy convictions into the daylight, where one can see whether they have legs or are merely opinions. Several chapters later — at this point we may admit that *several* is doing some heavy lifting — what began as a notebook had become something with a structure and an argument, and it seemed a waste not to share it.

The argument, such as it is, is built around a pattern. The pattern goes like this. Months one through three of any new codebase are a delight. The code is small, the choices are obvious, the team is moving fast, and everyone is in love with the project. Months five and six are still pretty good, with the first wave of feature requests landing on a foundation that — broadly — still fits. And then somewhere around month eighteen one wakes up to find the same codebase has acquired a faint chemical smell. Velocity has halved. Every feature now requires three meetings to scope. New starters look at it with the polite expression of someone being shown a baby they suspect is ugly.

What happened in between?

> *In nine cases out of ten, what happened in between was the model problem — a sequence of small, individually reasonable shortcuts that each, in isolation, looked like the right call on a Tuesday afternoon and that, together, became load-bearing.*

I should be clear: I do not believe most of those shortcuts were taken by careless people. Quite the reverse — they were almost all taken by careful people doing their best. The shortcuts compound. The walls accrete. The shape becomes the shape, and the shape is hard to refactor away from because the shape is now holding the roof up.

This book is my best attempt to identify the decisions whose long shadows you don't notice until they reach you. Some of those decisions are about types. Some are about models. Some are about whether to write a `try/catch`. Some are about how you store a date. Done well — and this is the thread I should mention at the outset, because it surprises people — these same choices also tend to be *faster*. Not the kind of "faster" that requires a profiler to detect; the kind that shows up as a snappier page, a lower cloud bill, an easier on-call shift. Doing this properly does not, as the conventional wisdom sometimes implies, mean sacrificing performance for tidiness. The tidy version is, very often, the quick version. I will keep pointing this out.

I should also be clear about what this is not. It is not a beginner's tutorial. It assumes you know what a class is and have, at some point, written `await` followed immediately by `.Result` and then quietly deleted it before anyone saw. It is not a reference manual; the documentation is excellent and I am not in competition with it. And it is not, I hope, a sermon. I have my opinions. I will tell you what they are and why. If you have found something cleaner, I would genuinely like to hear about it — there is a reading list at the back and a heart full of curiosity at the front.

Thank you for reading. Let us begin.
