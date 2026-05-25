# Chapter 39 — A Case Study: A Document-Intelligence Tool, Built Twice

We have spent thirty-eight chapters on the pieces. This chapter takes the whole toolkit out for a single drive.

The problem we will solve is one close to this book's own heart: *a knowledge-base tool that ingests a body of documents, understands them, and makes them queryable by an AI agent over the Model Context Protocol.* It is, in other words, the kind of tool you might build to make a collection of documentation — this very book, for instance — answerable to a question posed in plain language. It is modern, it is genuinely useful, and — importantly for our purposes — it has exactly the shape that makes the central question of this book worth asking: it begins simple, it accretes capabilities relentlessly, and at some point it either scales gracefully or it does not.

We will build it *twice* — once as a traditional model-based application, once as an ECS-based one — and we will watch what happens to each over a simulated eighteen months. Both will be built *well*: every good practice from the preceding chapters applied to each, no straw men. At the end, we will measure the one thing this book has argued matters most, and hand the verdict to you.

## The companion repository

This chapter is the narrative; the runnable proof is a companion repository. It contains both applications, deployable to your own Azure environment, with the milestones tagged in git so you can check out any point in the simulated timeline and see the code as it stood. The *"Running it yourself"* section near the end has the clone-configure-deploy steps; the repository's own README has the full detail. Everything in this chapter corresponds to real, deployable code you can run, change, and measure for yourself — because *"trust me, ECS scales better"* is exactly the kind of unsupported claim this book has tried throughout not to make.

## Month one — the honeymoon

The requirement is small. *Ingest a markdown document, split it into chunks, embed each chunk, store the vectors, and expose a `search` tool over MCP that returns the most relevant chunks for a query.*

**The model version.** A clean domain model, with the value-object and identifier disciplines of Part II already in place:

```csharp
public class Document
{
    public DocumentId Id { get; private set; }
    public DocumentPublicId PublicId { get; private set; }
    public DocumentSource Source { get; private set; }
    public string Content { get; private set; }
    public IReadOnlyList<Chunk> Chunks { get; private set; } = [];

    public void SetChunks(IReadOnlyList<Chunk> chunks) => Chunks = chunks;
}

public record Chunk(int Ordinal, string Text, ReadOnlyMemory<float> Embedding);
```

…with an `IngestionService`, a `RetrievalService`, an `IEntityTypeConfiguration<Document>`, and an MCP endpoint exposing `search`. Tidy, comprehensible, correct.

**The ECS version.** The document is an entity; its facets are components:

```csharp
public record struct Source(string Uri, DocumentPublicId PublicId);
public record struct RawContent(string Text);
public record struct Chunked(IReadOnlyList<string> Chunks);
public record struct Embedded(IReadOnlyList<ReadOnlyMemory<float>> Vectors);
```

…with a `ChunkingSystem` (operates on entities with `RawContent`, produces `Chunked`), an `EmbeddingSystem` (operates on `Chunked`, produces `Embedded`), and a `RetrievalSystem` (operates on `Embedded`). The MCP `search` tool invokes the retrieval system.

At month one, the two are *comparable in size and clarity*. If anything, the model version is marginally simpler — one class, a couple of services. The ECS version's component-and-system structure is a small amount of extra ceremony for no visible benefit yet. **If the project ended here, the model would be the right choice, and I would say so.** It does not end here.

## Month six — still holding up

Three requirements arrive over the following months, each reasonable, each small-sounding.

- **Multiple source types.** Not just markdown — PDFs, web pages, source-code files, each needing its own extraction.
- **Metadata filtering.** Retrieval should be filterable by source type, ingestion date, and tags.
- **Access control.** Documents have a visibility scope; retrieval must respect it.

**The model version.** `Document` grows. It gains a `SourceType`, a `Tags` collection, an `AccessPolicy`, an `IngestedAt`. New `PdfExtractor`, `WebExtractor`, `CodeExtractor` classes appear behind an `IExtractor` abstraction. The `RetrievalService` query gains filter parameters. The DTOs that serialise a document grow to match. It is all still *manageable* — the `Document` class is perhaps eighty lines now, the changes were mechanical, the tests went green. A good team feels no particular pain at month six.

**The ECS version.** Three new components — `SourceType`, `Tags`, `AccessPolicy` — and three new systems: `PdfExtractionSystem`, `WebExtractionSystem`, `CodeExtractionSystem`, each producing the same `RawContent` the chunking system already consumes. An `AccessFilterSystem` excludes entities the caller may not see. The existing `Document` entity definition does not change — there is no `Document` class to change. The existing chunking, embedding, and retrieval systems do not change — they still operate on `RawContent`, `Chunked`, and `Embedded` exactly as before.

**The diff at month six is close.** The model touched more files (the `Document` class, the DTOs, the query); the ECS version added more files (the new systems) but modified almost none. Reasonable people could call it a draw. *This honesty matters:* if the ECS advantage were already dramatic at month six, the book would be overselling it. It is not. The divergence is only beginning.

## Month eighteen — the wall

Now the requirements that a successful product accretes. Over the second year, the following arrive:

- **Per-tenant custom enrichers** — each client wants their own metadata extracted from their own documents.
- **Re-embedding and freshness policies** — documents go stale and must be re-embedded, possibly with a newer embedding model than the one that first processed them.
- **Multiple embedding models** in flight at once, during migrations.
- **PII redaction** — sensitive content stripped before embedding.
- **Summary generation** — an LLM-generated précis per document, itself embedded.
- **Hybrid retrieval** — combining vector similarity with keyword match and recency.
- **Provenance and citation tracking** — every retrieved chunk traceable to its source location.
- **Audit** — who ingested what, when, and under what policy.

**The model version hits the wall, exactly as [Chapter 15](../02_TheModelQuestion/15_TheWall.md) described.** The `Document` class is now past three hundred lines and climbing. It carries fields that are meaningful only sometimes (`RedactedContent` is null unless redaction ran; `Summary` is null unless summarisation ran; `ReEmbeddedAt` is null unless the freshness policy fired) — the conditionally-meaningful-nullable smell of [Chapter 5](../01_TheToolsAtHand/05_DiscriminatedUnions.md), multiplied. Adding the eleventh source type now touches: the `Document` class, the `SourceType` enum, the extractor factory, four DTOs, the retrieval query, the MCP tool schema, the migration, and the tests — *nine files*, for a concept the client described in one sentence. The per-tenant custom enrichers have nowhere clean to live, because the `Document` class is a single shape and the enrichers vary per tenant; the team ends up with a `Dictionary<string, object> CustomMetadata` bag on the document — the prototype-shaped C# of [Chapter 7](../01_TheToolsAtHand/07_BlueprintsVsPrototypes.md), the very thing that chapter warned against — and a thicket of `if (tenant == ...)` branches that [Chapter 37](../06_CodeThatDoesntSmell/37_Branching.md) catalogued as the silent killer. The team has not been careless. They have been overtaken.

**The ECS version stays additive.** Each new capability is a component and a system, and nothing else:

```csharp
// PII redaction — a component and a system. The Document "entity" is untouched.
public record struct Redaction(string RedactedText, IReadOnlyList<PiiSpan> Removed);
public class RedactionSystem { /* operates on RawContent, produces Redaction */ }

// Freshness — re-embed stale entities, regardless of what kind they are.
public record struct Freshness(DateTimeOffset LastEmbedded, EmbeddingModel Model);
public class ReEmbeddingSystem { /* operates on Freshness + Embedded, refreshes the stale */ }

// Summary — generated, then embedded like anything else.
public record struct Summary(string Text);
public class SummarizationSystem { /* operates on RawContent, produces Summary */ }
```

Per-tenant custom enrichers are tenant-specific systems, registered at runtime (the runtime-extensibility property of [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md)) — a tenant's enricher is a system that reads the components it cares about and writes a tenant-specific component, touching no shared code. The eleventh source type is one new extraction system emitting the standard `RawContent`. Audit, provenance, redaction, summary, freshness — each is a component plus a system, attached to whichever entities carry it, processed by whichever systems care, exactly as [Chapter 18](../03_ADifferentWay/18_MetadataAsAComponent.md) promised. The existing code is, almost entirely, not touched.

## The headline metric — the cost of change

This is the measurement the whole book has been building towards, and it is one you can reproduce yourself from the companion repository:

```bash
# In each application's repo:
git diff --stat month-06 month-18
```

The numbers you will see are, of course, specific to the implementation, so I will describe their *shape* rather than quote them as if they were laws of nature. What the diff reveals is this:

- **The model application:** a large change spread across many files, with substantial *modification* of existing code — the `Document` class rewritten, the DTOs reworked, the query rebuilt, the extractor factory extended. Many files changed; significant additions *and significant deletions*, because existing code had to be reshaped to accommodate the new requirements.
- **The ECS application:** a change concentrated in *new* files — new components, new systems — with existing files barely touched. Many additions; *deletions close to zero*, because nothing existing needed reshaping.

That near-zero deletion count in the ECS diff is the entire thesis of Part III, rendered as a git statistic. *Additive extension* is not a turn of phrase; it is a measurable property, and the diff between two milestones measures it. The cost of adding the second year's features is, in the model application, paid partly in *rewriting the first year's code*; in the ECS application, it is paid almost entirely in *new code that leaves the first year's alone*.

This is the metric, and it is the honest one. It is not a benchmark that can be rigged by a sympathetic load test. It is the literal diff of the literal code, and you can run it yourself.

## What "done poorly" looks like — on both sides

I have shown both architectures *done well*. Honesty requires showing what each looks like *done poorly*, because the well-vs-well comparison above is fair only if you know that both can be botched.

**The model, done poorly,** arrives at the wall much sooner — by month six rather than month eighteen. It is the `Document` class with public setters mutated from forty places, the persistence attributes smeared across the domain ([Chapter 10](../02_TheModelQuestion/10_WhatIsAModel.md)), the `string sourceType` instead of a value object, the N+1 retrieval that loads every document to filter in memory ([Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md)). The well-done model *delayed* the wall to month eighteen through discipline; the poorly-done one hits it in the first quarter. The discipline of Parts II and IV buys you time — it does not, on its own, change the destination.

**The ECS version, done poorly,** can be *worse than a good model*, and this is the warning [Chapter 21](../03_ADifferentWay/21_ECSSmells.md) exists to give. Bloated components that have drifted back into being models (a `Document` component with twenty fields — the very class we were trying to escape, now wearing a component's clothes). Stateful systems with hidden caches that destroy the testability and parallelism that were the point. Over-componentisation that scatters a simple change across twelve files. An ECS codebase without the discipline of flat components and stateless systems combines the unfamiliarity of ECS with the bloat of models, and arrives somewhere genuinely unpleasant. *ECS rewards conviction and punishes half-heartedness*, exactly as the reckoning chapter warned.

So the 2×2 is real, and the comparison that matters is the diagonal — *well* against *well* — with both *poorly* corners standing as honest cautions rather than rigged contrasts.

## The honest secondary benchmark — retrieval performance

The companion repository also includes a load-test trigger that runs identical queries over identical seed data against both applications, and reports retrieval latency. I include it because performance is measurable and the reader will want to see it — but I want to be precise about what it does and does not show.

It shows the *data-access* difference of [Chapter 27](../04_DataAndSets/27_LinqAndEfCore.md) and [Chapter 30](../04_DataAndSets/30_PersistingECS.md): the well-done model, by month eighteen, tends to materialise fat `Document` entities to answer a query that needs three fields, while the ECS version projects only the components retrieval requires. So the ECS retrieval path is typically faster — but the honest attribution is that *it is faster because it materialises less, not because ECS is intrinsically quicker*. A model application equally disciplined about projection would close most of the gap.

What the benchmark *cannot* show — because both applications are deployed as request-scoped Azure Functions that reload state per invocation — is ECS's headline raw-throughput advantage, the cache-coherent in-memory iteration of [Chapter 19](../03_ADifferentWay/19_Registries.md). That advantage belongs to long-lived stateful processes, and a function app is not one. I mention this so that you do not over-read the benchmark in *either* direction: it is a fair measure of the data-access patterns, and a silent witness on the in-memory story. As [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md) said plainly, *performance is not the reason to choose ECS for business software* — the cost-of-change diff above is.

## Running it yourself

The companion repository is built to be cloned, configured, and deployed to your own Azure environment, so that none of the claims above need be taken on trust.

1. **Clone** the repository and open the solution. It contains two deployable applications — `KnowledgeBase.Model` and `KnowledgeBase.Ecs` — and a shared `seed/` directory of test documents.
2. **Configure** by copying `appsettings.template.json` to `appsettings.local.json` in each app and filling in: your embedding endpoint and key (Azure OpenAI or a local model), your vector-store connection, and your storage connection. The README lists every field.
3. **Deploy** with the supplied `azd` (Azure Developer CLI) configuration: `azd up` provisions the function apps, the storage, and the vector store, and deploys both applications. The single config drives both, so they run against identical infrastructure.
4. **Seed** by running the `seed` trigger on each app, which ingests the shared test documents — the same corpus into both, so the comparison is like-for-like.
5. **Measure.** Run the `benchmark` trigger to compare retrieval latency, and — the headline — run `git diff --stat month-06 month-18` in each app to see the cost-of-change difference in black and white.

The whole point of the exercise is that you do not have to believe me. You can deploy both, point an MCP-capable agent at each, ask the same questions, watch the same numbers, and read the same diffs. The corpus can be anything — including, pleasingly, the markdown of this book.

## The verdict — handed back, one last time

I am not going to tell you which one won.

You have read Part II and Part III. You have now seen the same problem built both ways, both built well, with the failure modes of each laid out honestly, and with the one metric that matters reproducible on your own machine. You know your system, your team, and your eighteen-month roadmap better than I do. If your knowledge-base tool will stay small and stable, the model version is the simpler, friendlier choice, and the diff at month six — where the two are close — is the relevant one for you. If it will grow the way successful products grow, the diff at month eighteen is the one to look at, and it says what it says.

What I will do, as I did in [Chapter 22](../03_ADifferentWay/22_ModelsVsECS.md), is decline to pretend the choice is universal, and point one more time at the thing the case study was built to make visible: *the cost of the second year is paid differently by the two architectures, and the difference is not a matter of taste — it is a number you can read off a git diff.*

Where you go from there is, as it should be, up to you.

## Recap

- The case study builds the same document-intelligence MCP tool twice — model and ECS — both done well, over a simulated eighteen months.
- At month one the model is marginally simpler; at month six the two are close; at month eighteen they diverge sharply.
- The headline metric is `git diff --stat` between milestones: the ECS application's near-zero *deletions* are *additive extension* rendered as a statistic.
- Both architectures done poorly are shown as honest cautions — a bad model hits the wall sooner, a bad ECS can be worse than a good model.
- The secondary performance benchmark fairly measures data-access patterns and is honestly silent on ECS's in-memory advantage, which a function app cannot exhibit.
- The verdict is handed to the reader, with one reproducible number to inform it.

## Onwards

One chapter remains — a short one. A look back at what the series tried to do, an honest accounting of what it deliberately left out, and a few words on where to go next. Then we are done.
