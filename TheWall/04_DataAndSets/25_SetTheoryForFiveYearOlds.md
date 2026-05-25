# Chapter 25 — Set Theory for Five-Year-Olds

I want, in this chapter, to teach you the entire mathematical foundation of every database query you will ever write. It will take about fifteen minutes. By the end of it, you will understand `WHERE`, `JOIN`, `GROUP BY`, `DISTINCT`, `UNION`, and `EXCEPT` better than most working developers, and you will understand the LINQ method names of [Chapter 26](./26_FiltersJoinsGroupings.md) so naturally that you may wonder why anyone considered them unintuitive in the first place.

The good news is that you already understand the entire mathematical foundation. You have done so since you were about three years old. You just have not, until this minute, had anyone tell you that this is what it is called.

The branch of mathematics is *set theory*. We will, in time, get to formal definitions. We will start with a dog.

## Spaniels, dogs, animals

Imagine a basket on the floor. In the basket are three spaniels. Their names are Ada, Beau, and Cleo. Reach into the basket. Pull out a spaniel. You have a spaniel.

The basket of spaniels is a *set*. A set is a collection of things — *members*, *elements* — that we have grouped together. The basket contains exactly Ada, Beau, and Cleo. It does not contain anything else. There are no duplicates (Ada appears once). The order in which we listed them does not matter (it is the same basket whether we said *Ada, Beau, Cleo* or *Cleo, Ada, Beau*).

In notation, mathematicians write it like this: **{Ada, Beau, Cleo}**. Curly braces. Comma-separated. That is the entire syntax. You now know set notation.

Now imagine a slightly bigger basket. This basket is called *Dogs*. It contains Ada, Beau, and Cleo, but also Rex (a retriever), Lola (a collie), and a great many others. The basket of spaniels and the basket of dogs *overlap* — every spaniel is also a dog, but not every dog is a spaniel.

We say *the set of spaniels is a **subset** of the set of dogs*. Every member of the smaller set is also a member of the larger set. The larger set is a *superset* of the smaller one. There is, gratifyingly, nothing more to it.

Bigger still: the set of *Animals*. Contains all the dogs. Contains all the cats. Contains all the wombats, the parrots, the otters, the slightly unhappy iguanas. Dogs are a subset of animals. Spaniels are a subset of dogs are a subset of animals. The nesting could continue all the way out to *everything in the universe*, which is a set so large mathematicians have given it a special name and tend to avoid it on philosophical grounds.

## The six things you can do with sets

There are precisely six operations on sets that you will, in practice, use most days of your working life. None of them is hard.

### One — union

Suppose I have a set of *spaniels* (Ada, Beau, Cleo) and a set of *retrievers* (Rex, Sam, Tess). I want one big set containing all of them.

```
{Ada, Beau, Cleo} ∪ {Rex, Sam, Tess} = {Ada, Beau, Cleo, Rex, Sam, Tess}
```

The little cup symbol — `∪` — is *union*. *"Everything that is in either set."* Note that no duplicates appear in the result. If Beau happened to be in both sets (perhaps he is a spaniel-retriever cross), the union would still contain only one Beau. *Sets do not have duplicates, by definition.*

### Two — intersection

Suppose I have a set of *good boys* (Ada, Beau, Rex, Tess) and a set of *spaniels* (Ada, Beau, Cleo). I want the set of *spaniels who are also good boys*.

```
{Ada, Beau, Rex, Tess} ∩ {Ada, Beau, Cleo} = {Ada, Beau}
```

The little cap symbol — `∩` — is *intersection*. *"Everything that is in both sets."* Ada is in both, so she is in the result. Cleo is a spaniel but not, on today's evidence, a good boy. Rex is a good boy but not a spaniel. Both fall away.

### Three — difference

Suppose I want the spaniels who are *not* good boys.

```
{Ada, Beau, Cleo} \ {Ada, Beau, Rex, Tess} = {Cleo}
```

The backslash — `\` — is *difference*, sometimes called *minus* or *exclusion*. *"Everything that is in the first set but not in the second."* The order matters: `A \ B` is not the same as `B \ A`. Cleo is the only spaniel who has yet to make the *good boys* list. Tomorrow, perhaps.

### Four — complement

Suppose I want everything in the universe that is *not* a spaniel.

```
ℂ({Ada, Beau, Cleo}) = {Rex, Sam, Tess, all wombats, the Queen, ...}
```

The complement is *"everything not in the set"*. In practice, this is always taken relative to some agreed *universe* — *"the set of dogs that are not spaniels"*, *"the set of customers who have not placed an order"*. Complement on its own is rarely useful; complement against a stated universe is one of the most common queries in business software.

### Five — subset

The relationship *"every member of A is also a member of B"*. Spaniels are a subset of dogs. Dogs are a subset of mammals. Programming questions phrased as *"are all of these also that?"* are subset questions.

### Six — Cartesian product

The most important operation for our purposes, and the one with the slightly intimidating name. The Cartesian product of two sets is *every possible pairing of one element from each*.

```
{Ada, Beau} × {biscuit, sock} = {(Ada, biscuit), (Ada, sock), (Beau, biscuit), (Beau, sock)}
```

If I have two spaniels and two possible objects to give them, the Cartesian product is the four possible *gifting events* — Ada with a biscuit, Ada with a sock, Beau with a biscuit, Beau with a sock. Four pairings. If I had three spaniels and four possible objects, there would be twelve. *The size of the Cartesian product is the product of the sizes of the input sets.* This is, you will note, where the operation gets its name.

Hold on to the Cartesian product. We will come back to it in a moment.

## The two things you can do *to* sets

Sets have two other operations that, strictly speaking, are not *between* sets but *over* them.

**Filter.** *"Give me only the members that satisfy this condition."* From the set of spaniels, give me the ones whose name starts with B. From the set of orders, give me the ones placed last Tuesday. The filter operation produces a *subset* of the original set.

**Map (or projection).** *"For each member, produce a different value derived from it."* From the set of spaniels, produce the set of their names. From the set of orders, produce the set of order totals. The result is a set of the same size (or smaller, if duplicates collapse) containing derived values.

That is it. Filter and map. Combined with the six set operations above, these eight ideas cover every data-querying operation in the working developer's daily life. There are no others.

## The reveal

I promised you, at the start, that you already understand the foundation of every database query. Let me cash the cheque.

Every `WHERE` you have ever written is a **filter**.

Every `SELECT col1, col2` is a **map** — taking the original set of rows and projecting only the fields you want.

Every `JOIN` is a **Cartesian product followed by a filter** — take every possible pairing of rows from the two tables, then keep only the pairings where the join condition is true. A `JOIN ... ON a.id = b.aid` is *"the Cartesian product of A and B, filtered to keep only pairs whose IDs match."* The database does this efficiently with indexes; conceptually, it is still a filtered Cartesian product.

Every `GROUP BY` is a **partition** — splitting the set into smaller sets that share a common value, then producing one row of aggregated output per smaller set.

Every `DISTINCT` is the recognition that *sets do not have duplicates, by definition*, applied to a result that has been allowed to collect duplicates along the way.

Every `UNION` is, mercifully, a **union**.

Every `EXCEPT` is a **difference**.

Every `INTERSECT` is, by the time you have got this far, exactly what you think it is.

This is not a coincidence. Relational databases were invented, in the 1970s, by a mathematician (Edgar Codd) who was deliberately translating the operations of set theory into a query language that working programmers could use. *Every query you have ever written is set theory wearing a high-vis vest.* Once you see it, you cannot unsee it, and your queries will, very quietly, get better.

## Performance, in passing

A small bonus. Once you see your data work as set operations, the complexity classes of each operation become visible.

A filter is O(n) — you touch each element once. A map is O(n). A union is O(n + m). An intersection, with a hash-based index, is O(min(n, m)). A Cartesian product is O(n × m) — which is why a `JOIN` without a good index is, sometimes, catastrophically slow: the database is, in effect, building the full pairing set before filtering it. The presence of an index turns the Cartesian-product-then-filter into something closer to a lookup, which is the whole reason indexes exist.

Knowing the set-theoretic shape of a query tells you, before you have run it, what its cost will look like. Knowing the cost tells you, before you have written it, what indexes you will need. This is one of the reasons working developers who understand set theory write queries that are quietly faster than working developers who do not.

> *Most database performance problems, examined closely, turn out to be set operations being performed without the indexes that would let them dodge the Cartesian product. The fix is rarely a faster query; the fix is to stop performing the wrong operation by accident.*

## Recap

- A set is a collection of distinct things in no particular order.
- The six operations between sets are union, intersection, difference, complement, subset, and Cartesian product.
- The two operations *over* a set are filter and map.
- Every data query you have ever written is one or more of these in disguise.
- Indexes exist to dodge the cost of the Cartesian product. Most database performance problems are this principle in slow motion.

## Onwards

The next chapter makes the promise of this one fully explicit. We will take a small set of LINQ method names — `Where`, `Select`, `SelectMany`, `Join`, `GroupBy`, `Distinct`, `Union` — and show that each one is exactly one of the operations above, with a C# accent. By the end of it, the LINQ method names will, I hope, feel less like an arbitrary vocabulary and more like the natural English (or near-English) translations of the eight ideas you have just internalised.
