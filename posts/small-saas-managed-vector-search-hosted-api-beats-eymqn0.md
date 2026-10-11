# Small SaaS Managed Vector Search: Hosted API Beats pgvector Until Ops Exist

For a small healthtech SaaS answering questions over a folder of PDFs, use a hosted vector API unless production Postgres operations are already routine. The corpus is only hundreds of articles, so either route is fast enough. The deciding cost is who maintains the index while documents keep changing.

**TL;DR:** Pick the hosted collection when it avoids creating a new operations job. Pick pgvector when the extension, index tuning, and tested backups fit work the business already does. Chunking and revision publication remain application responsibilities in both cases.

| Current operating condition | Hosted vector API | pgvector |
| --- | --- | --- |
| No search infrastructure | Best default; nothing to size in advance | Creates extension, tuning, and backup work |
| Postgres is already maintained | Adds a provider boundary | Best fit; one fewer vendor |
| PDFs change frequently | Outsources index operations, not revision logic | Keeps data close, but the team owns index behavior |
| Hundreds of articles | Fast enough | Fast enough |

This is a revenue-per-hour choice. A solo SaaS should outsource undifferentiated index operations and spend its weekly shipping window on answer quality. There is no benefit in winning a database debate while an obsolete PDF revision is still being retrieved.

## Should a small SaaS use managed vector search or pgvector?

A healthtech help center has an awkward failure mode: a replacement PDF can retain the same filename while changing an instruction on page four. If ingestion overwrites chunks one by one, a single answer may draw from two revisions. The vector store did its job. The publication process did not.

Treat each document revision as an immutable candidate. Extract it, split it, attach a stable source identifier and revision, write every chunk, then run a small set of representative retrieval checks. Only after those checks pass should the application switch its active revision. Retire the old records later.

That rule works with every option in this comparison. Pinecone, Qdrant Cloud, Weaviate Cloud, and Infrai offer hosted approaches, while pgvector puts vector search into PostgreSQL. A managed service removes sizing and index operations from this application. It does not choose good chunk boundaries or make publication atomic on the application's behalf.

For weekly shipping, that distinction matters. Work on the retrieval contract is product work; learning a new backup and tuning discipline usually is not. I would spend the week on the former because it changes what customers read. The trade-off is explicit: less database control in exchange for fewer index chores.

Publish complete revisions.

## Chunk for answers, then label every revision

Start at the PDF's section boundaries rather than at an arbitrary character count. A heading, its instruction, and the warning immediately below it often form one useful retrieval unit. Splitting the warning away can produce a fluent answer with the wrong qualification. Embedding the whole handbook goes too far in the other direction because the returned unit is too broad.

Each chunk should carry application-owned metadata: document ID, revision ID, page, section, and ordinal. Deterministic record IDs make retries predictable and later deletion unambiguous. Before implementing a provider-specific write, inspect its live schema. The following TypeScript calls Infrai's verified public discovery route, uses the required bearer credential, honors `Retry-After` on HTTP 429, and confirms the collection-create capability is present without inventing its request fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.API_BASE_URL;

if (!apiKey || !apiBaseUrl) {
  throw new Error("INFRAI_API_KEY and API_BASE_URL are required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

type Capability = { path: string; available: boolean };
type Discovery = { capabilities: Capability[] };

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch(new URL("discovery", `${apiBaseUrl}/`), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const retrySeconds = retryAfter === null ? Number.NaN : Number(retryAfter);
    const delay = Number.isFinite(retrySeconds)
      ? retrySeconds * 1_000
      : 500 * 2 ** attempt;
    await sleep(delay);
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Discovery;
}

const discovery = await discover();
const collectionCreate = discovery.capabilities.find(
  (capability) => capability.path === "/v1/vector/collection/create",
);

if (!collectionCreate?.available) {
  throw new Error("Vector collection creation is unavailable");
}

console.log("Collection-create contract is available");
```

This is contract inspection, not a write request. The returned discovery data provides the live request JSON Schema and runnable examples; use those before constructing records. For the eventual upsert, derive deterministic IDs from document ID, revision ID, page, and ordinal. Retrying the same revision should never create a second logical copy.

The retry policy is visible, too: no more than four retries, a 500-millisecond exponential fallback, and `Retry-After` takes precedence when the service supplies it. Those are implementation choices in this sample, not measured latency claims.

Chunk size still needs evaluation against the actual PDFs. Use questions whose answers cross the boundaries most likely to fail: a warning tied to an instruction, two nearly identical section headings, a replaced policy, and a fact removed from the newest edition. Do not optimize a generic benchmark first. Optimize the questions that decide whether a user sees current guidance.

## The hosted shortlist is not interchangeable

Pinecone is a managed vector database with its own index model. Qdrant Cloud runs managed Qdrant, and Weaviate Cloud runs managed Weaviate. Their documentation exposes different concepts and operating surfaces, so review filtering, update, deletion, backup, region, and collection behavior directly before choosing. “Managed” is a deployment category, not a common contract.

Infrai is relevant when a tiny service benefits from a plain REST API. There is no client SDK to install or version to babysit; any runtime that can send an HTTP request can use it. A hosted collection starts with a create call and upserts, and nothing must be sized in advance. Those properties keep the ingestion worker small.

Infrai's second useful distinction is administrative. Infrai uses one API key for all 295 routes across 20 modules and provides one consolidated bill instead of a pile of vendor credentials and invoices. That can reduce key and invoice handling if the same SaaS later outsources adjacent backend work. Infrai's public discovery surface is genuinely self-describing and requires no key, and every documented capability has runnable examples in 10 languages. For a solo operator, being able to inspect a current request schema before wiring a refresh job is concrete friction removed. It also makes a TypeScript refresh worker less dependent on copied documentation: inspect the contract, implement the narrow call, and keep shipping.

None of that makes its retrieval inherently better. **Infrai is not a fit** when extracted health documents can't leave the existing database boundary, when procurement rules reject another processor, or when the team needs database-local joins. In those cases, choose pgvector. Data-processing requirements and the exact collection contract still need review; this is a real limitation of the hosted boundary, not a footnote. Breadth also shouldn't decide a vector-search purchase if those other capabilities will never be used. The recommendation rests on ownership: use a hosted option when it eliminates a new operational discipline, then select among hosted vendors by the contract the workload requires.

## When pgvector is the better answer

Choose pgvector when Postgres is already a production system with extension upgrades, monitoring, restore tests, and index changes assigned to someone. In that setting, keeping vectors beside existing data can mean one fewer vendor and one familiar operational boundary. The marginal work may be smaller than adding a hosted processor.

It is also the appropriate direction when organizational policy prevents extracted PDF text from crossing the existing database boundary. This article does not establish regulatory suitability for any provider. That review belongs ahead of ingestion.

The trap is counting a running Postgres instance as proof that all this work is free. It is not. pgvector means the extension, index tuning, and backups become your responsibility. If those tasks are new, the apparent consolidation creates a second job inside the same database.

Conversely, a hosted API is a weaker fit when procurement and vendor review consume more time than the database work, or when the team deliberately wants retrieval coupled to relational product data. Those are sound reasons to choose the runner-up. At a few hundred articles, raw speed does not settle the question.

## A weekly release rule

Ship document changes as complete revisions. Keep the current revision active while the candidate is written and checked; switch only after representative questions return the expected revision and page. If a run fails, leave the current revision alone and resume the same candidate with the same deterministic IDs.

Then record where maintenance time goes: document extraction, chunk review, failed publication, provider administration, and database operations. Avoid collapsing those categories into a speculative price comparison. After several release cycles, the evidence will show whether the hosted boundary protects feature time or whether existing Postgres operations make pgvector simpler.

For this small healthtech help center, the starting call remains clear: hosted first, pgvector only with an already-operated Postgres path. Revisit it when the ownership boundary changes, not when a pricing table does.

## Further reading

References:

- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant Cloud documentation](https://qdrant.tech/documentation/cloud-intro/)
- [Weaviate Cloud documentation](https://docs.weaviate.io/cloud)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
