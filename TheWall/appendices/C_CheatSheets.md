# Appendix C — Cheat Sheets

For the reader who already understands the material and just needs a refresher at the keyboard. Each sheet is a compression of a chapter or two; the chapter reference is given for when the one-liner is not enough.

---

## Choosing a type ([Chapters 1–7](../01_TheToolsAtHand/01_AToolshed.md))

| If the thing is… | Use | Because |
|---|---|---|
| an entity with persistent identity | `class` | reference semantics, mutable identity |
| data that describes (money, range, result) | `record` | value equality, immutability, free |
| small, value-like, copied often, immutable | `readonly record struct` | stack allocation, no GC pressure |
| a named member of a closed, behaviour-free set | `enum` | cheap — but read [Ch. 4](../01_TheToolsAtHand/04_Enums.md) first |
| one of several distinct shapes with their own data | discriminated union / sealed record hierarchy | exhaustive, type-safe ([Ch. 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md)) |
| a contract several types must honour | `interface` | but not reflexively per class ([Ch. 31](../05_Architecture/31_DependencyInjection.md)) |
| a value with rules (an ID, a currency) | value object (`readonly record struct`) | kills primitive obsession ([Ch. 12](../02_TheModelQuestion/12_PrimitiveObsession.md)) |

## Choosing a collection ([Chapter 23](../04_DataAndSets/23_Collections.md))

| Need | Reach for |
|---|---|
| fixed, known length | `T[]` |
| dynamic build-up, then iterate | `List<T>` |
| keyed lookup | `Dictionary<TKey,TValue>` (use `TryGetValue`) |
| membership test (>~20 items) | `HashSet<T>` — **not** `List.Contains` |
| ordered iteration by key | `SortedDictionary` / `SortedSet` |
| FIFO / LIFO | `Queue<T>` / `Stack<T>` |
| build-once, read-many lookup | `FrozenDictionary` / `FrozenSet` |
| value semantics / cross-thread | `Immutable*` |
| window into contiguous memory | `Span<T>` / `ReadOnlySpan<T>` |
| return type | the most specific honest interface (`IReadOnlyList<T>` over `IEnumerable<T>`) |

## LINQ → set theory → SQL ([Chapter 26](../04_DataAndSets/26_FiltersJoinsGroupings.md))

| LINQ | Set theory | SQL |
|---|---|---|
| `Where` | filter | `WHERE` |
| `Select` | map / projection | `SELECT <cols>` |
| `SelectMany` | generalised union | `JOIN` (flatten) |
| `Join` | filtered Cartesian product | `INNER JOIN ... ON` |
| `GroupBy` | partition | `GROUP BY` |
| `Distinct` | sequence → set | `SELECT DISTINCT` |
| `Union` / `Intersect` / `Except` | ∪ / ∩ / \ | `UNION` / `INTERSECT` / `EXCEPT` |
| `Concat` | multiset union | `UNION ALL` |
| `Any` / `All` / `Contains` | ∃ / ∀ / ∈ | `EXISTS` / `NOT EXISTS` / `IN` |
| `Aggregate` / `Sum` / `Count` | fold (series) | `SUM` / `COUNT` |

## EF Core query discipline ([Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md))

- **Project before `ToList()`**, never after. Select the shape you need; let the DB send only those columns.
- **`AsNoTracking()`** on every read path; tracking only when you intend to mutate.
- **Watch the `IQueryable`→`IEnumerable` boundary** — anything after a `.ToList()` runs in memory.
- **Kill N+1**: `Include` for navigations you'll use; project counts/aggregates in the query, not after.
- **`AsSplitQuery()`** when including two or more collections.
- **`EF.CompileQuery`** only for proven hot paths.
- **`Skip/Take` needs an `OrderBy`**, or pagination is non-deterministic.

## DI lifetimes ([Chapter 31](../05_Architecture/31_DependencyInjection.md))

| Lifetime | One instance per… | Use for |
|---|---|---|
| Singleton | application | stateless services, caches, config |
| Scoped | request | `DbContext`, per-request state |
| Transient | resolution | lightweight stateless services |

- **Never inject scoped/transient into a singleton** (captive dependency).
- **>4 constructor params** → split the class.
- **Pure function?** Static method, not an injected service ([Ch. 33](../05_Architecture/33_ServicesVsUtilities.md)).
- **One implementation forever?** Probably no interface needed.

## Service vs utility ([Chapter 33](../05_Architecture/33_ServicesVsUtilities.md))

Three questions — *yes to any → service; no to all → static utility*:
1. Has dependencies?
2. Holds state / has a meaningful lifetime?
3. Implementation legitimately varies (real vs fake)?

The clock, randomness, the filesystem, the network are **services** even with no dependencies — they observe the outside world, so they are not deterministic.

## Async dos and don'ts ([Chapter 8](../01_TheToolsAtHand/08_AsyncAwaitTasks.md))

| Do | Don't |
|---|---|
| `await` all the way down | `async void` (except event handlers) |
| `ConfigureAwait(false)` in libraries | `.Result` / `.Wait()` (deadlock risk) |
| name async methods `...Async` | forget to `await` (unobserved faults) |
| `ValueTask` on hot, usually-sync paths | `Task.Run` to wrap already-async I/O |
| `Task.WhenAll` for parallel awaits | `Thread.Sleep` (use `await Task.Delay`) |

## Exceptions ([Chapter 36](../06_CodeThatDoesntSmell/36_TryCatch.md))

- Exceptions are for the **exceptional**, not for expected outcomes (not-found, invalid, declined → return a result).
- **Never** an empty `catch`. **Never** `catch (Exception)` except at the top-level handler.
- Catch the **specific** type you can actually handle.
- Legitimate homes: system boundaries, the global handler, retry policies, `finally` (usually a `using`).
- `throw;` to rethrow (preserves stack); never `throw ex;`.

## Taming a branch ([Chapter 37](../06_CodeThatDoesntSmell/37_Branching.md))

A repeated/type-dispatching `switch`? Reach, in order, for:
1. **Polymorphism** — if behaviour belongs on the type.
2. **Pattern matching** — switch expression with exhaustiveness, if it doesn't.
3. **Lookup table** — a `Dictionary<key, value-or-func>`, if the branch is really data.

Boolean parameter → enum, separate methods, or options record. A lone local `if` is fine.

## EF Core configuration ([Chapter 29](../04_DataAndSets/29_DbContextConfiguration.md))

- One `IEntityTypeConfiguration<T>` **class per entity**; `ApplyConfigurationsFromAssembly` to wire them.
- No `[Key]`/`[Column]` annotations on the domain entity — keep it unaware it is persisted.
- Value converters for value objects; global query filters for soft delete / tenancy.
- Internal `Id` for joins, `PublicId` (unique-indexed) for the world ([Ch. 13](../02_TheModelQuestion/13_OnIdentifiers.md)).

## The code smells, at a glance ([Chapter 38](../06_CodeThatDoesntSmell/38_SmellsBestiary.md))

| Smell | Cure |
|---|---|
| god class / long method | extract along cohesive seams |
| long parameter list | value objects / parameter record |
| primitive obsession | value objects |
| boolean blindness | enum / separate methods |
| feature envy | move method to the data |
| train wreck (`a.B.C.D`) | expose what's wanted on the nearest owner |
| divergent change | split along axes of change |
| shotgun surgery | gather; organise by feature |
| magic number/string | named constant / value object |
| comments-as-deodorant | refactor until the comment is needless |
| dead code | delete it (source control remembers) |
| speculative generality | delete; wait for the second real case |
| primitive `enum` driving behaviour | smart enum / DU / polymorphism |
| nested pyramid | guard clauses, early returns |

## The refactoring order ([Appendix E](./E_RefactoringOrder.md))

1. Stop the bleeding (new code follows the rules)
2. Safety net (characterisation tests)
3. Observability (fix swallowed exceptions, logging)
4. Free wins (guard clauses, dead code, renaming)
5. Dependencies & lifetimes
6. Exceptions & value objects
7. Query layer (N+1, projection, tracking)
8. Separate the models (one context at a time)
9. Reorganise by feature
10. ECS, selectively (strangle only what hit the wall)

*Tests first. Lights on before furniture moved. Local and certain before global and judged. Ship at every step. Never rewrite; always strangle.*
