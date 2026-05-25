# The Wall
### Why software gets hard to change — and a different way to build
*by a Bricklayer*

> *Education optional. Ingenuity mandatory.*

These are the notes of someone who spends evenings and weekends thinking about why
perfectly good codebases get slow and painful to change — and who got tired enough
of the feeling to write it all down.

Up front, so there's no confusion about what this is: it is **not** authoritative.
I'm not selling a framework, a course, or a consultancy, and nobody appointed me to
tell you how to build software. I'm an armchair architect — these are my own opinions,
formed over a lot of late evenings and a small number of disasters. Strong opinions,
but held loosely. If you've found something better, I would genuinely like to hear it.

## The wall

Every reasonable team, on every reasonable Tuesday, lays another brick: a sensible
field here, a small new entity there, one more perfectly-defensible shortcut. Each
brick is fine. And then, somewhere around the eighteenth month, you look up and
realise the bricks have become a wall — every new feature takes longer than the last,
and no amount of tidying quite seems to lift the feeling.

That software decays like this is the oldest story in the field; it has famous names
— technical debt, software entropy, the big ball of mud. The thing I think is worth
saying is not *that* the wall exists. Everyone knows that. It is *why*, for a
particular and very common kind of system: the wall is **structural**, not a
discipline problem, and it has a specific cause you can point at. There is an
architecture in which the cost of laying the next brick *stays flat* — and, the part
I care about most, you can **measure the difference yourself** with a plain `git diff`.

All in all, it needn't be just another model in the wall.

## What's in here

- **The book** — a readable, opinionated tour: the C# type system, models and where
  they crack, a different way of structuring software, the set theory hiding under
  your queries, application architecture, and the daily habits that separate code
  you're proud of from code you keep quiet about at parties.
- **A runnable comparison** — the same small system built two ways, deployable to
  your own environment, so you can *see and measure* the difference rather than take
  my word for it.

## See for yourself

1. Clone the repo.
2. Copy `appsettings.template.json` to `appsettings.local.json` in each app and fill
   in the blanks (each app's README lists every field).
3. `azd up` to deploy both apps to your own environment.
4. Seed both with identical data, run the benchmark, and — the headline — run
   `git diff --stat month-06 month-18` in each app to watch the cost of change
   diverge in black and white.

## If you think I'm wrong

Brilliant. Open an issue and tell me where. That is the whole point — strong opinions,
held loosely, and a genuine wish to be shown something better.

— *The Bricklayer*
