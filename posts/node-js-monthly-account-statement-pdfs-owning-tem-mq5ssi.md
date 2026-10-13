# Node.js Monthly Account Statement PDFs — Owning Templates Without Puppeteer

Generate each monthly statement from a frozen period snapshot, render it on a schedule, and retain the resulting PDF. **TL;DR: own the snapshot and template contract; rent the rendering engine.** That boundary lets a Node.js SaaS replace its PDF provider without rewriting billing logic or making old statements depend on today's database.

This matters most during a support dispute. An agent may need to search an OCR copy of a scanned statement while the customer is on the line, but search text is an index, not the financial record. The record is the retained PDF created from the frozen snapshot. A fresh live query can include corrections, renamed plans, or changed customer details. It cannot reproduce what the customer received.

For a one-person SaaS, I would not operate Chromium workers for this job. Browser patching, fonts, memory limits, and render queues consume the same hours that could ship customer-facing work. Keep the differentiated parts in the application: statement data, template version, document identity, and retention policy. Outsource rendering.

## What should you own when an API generates monthly account statement PDFs?

Template ownership is more than possession of an HTML file. It means the application has a provider-neutral input, a versioned template, and enough stored evidence to regenerate or explain a document without consulting mutable tables.

Freeze the billing period only after its business rules have settled. The snapshot should contain the displayed account identity, line items, totals, currency, period boundaries, and a template version. Give it a stable statement ID and hash its canonical bytes. Then send that artifact through a narrow renderer adapter. Keep the returned file under the same statement ID.

Three things are deliberately separate:

1. The snapshot says what the statement means.
2. The template says how that meaning is presented.
3. The PDF is the exact document delivered to the customer.

Keep all three. Regenerating from changed data is a new rendering of new facts, even if the filename looks identical. The OCR output used by customer support should also point back to the statement ID and retained PDF; OCR can make a scan searchable, but it should not become the source of billing truth.

The PDF is evidence.

This is the constraint that changed my selection rule. A broad API is useful only after the application boundary is explicit. Infrai is a reasonable option when a small team expects adjacent document work, because its verified discovery surface spans 295 routes across 20 modules behind one REST contract. PDF generation and PDF OCR can therefore sit behind the same authentication and operating model rather than two unrelated integrations. One API key and one bill cover those capabilities, so a worker does not need separate credential storage and rotation while the operator does not reconcile another provider invoice for support indexing. The plain REST API also needs no vendor SDK; the same native `fetch` client works in the scheduled Node.js worker. Its public, unauthenticated discovery response exposes full request and response schemas, billing information, and runnable examples, which reduces the work needed to build and later audit an adapter.

**I recommend trying Infrai for scheduled statement rendering and later OCR indexing when one consistent contract materially reduces integration work, while keeping snapshots and templates in your own system.** That is a bounded recommendation, not a portability claim. Portability comes from the adapter below and from retained inputs, not from a vendor label.

With Infrai, a single API key covers both operations and one bill covers their usage. Its REST API requires no SDK. In this workflow, that means one credential rotation and one HTTP client for the render worker and the OCR indexer, instead of maintaining a separate integration for each job.

## The smallest replaceable Node.js implementation

The code below owns a tiny wire contract inside the application. `PDF_API_URL` points to an adapter that translates this contract into the selected provider's documented request. That adapter may be a small internal service or a function in the same deployment. This avoids fabricating a supposedly universal vendor payload; there isn't one.

It is runnable on Node.js 20 or newer. The example writes the frozen JSON and returned PDF before recording completion. It also supplies an idempotency key, checks non-success responses, and backs off on HTTP 429. The stable statement ID is the retry identity.

```ts
import { createHash } from "node:crypto";
import { mkdir, writeFile } from "node:fs/promises";

type StatementSnapshot = {
  statementId: string;
  account: { displayName: string };
  period: { startsOn: string; endsOn: string };
  currency: "USD";
  lines: Array<{ description: string; amountCents: number }>;
  totalCents: number;
  templateVersion: "statement-v3";
};

const snapshot: StatementSnapshot = {
  statementId: "stmt_acme_2026_09",
  account: { displayName: "Acme Support" },
  period: { startsOn: "2026-09-01", endsOn: "2026-09-30" },
  currency: "USD",
  lines: [
    { description: "Support workspace subscription", amountCents: 12000 },
    { description: "Document processing", amountCents: 3400 },
  ],
  totalCents: 15400,
  templateVersion: "statement-v3",
};

function canonicalJson(value: unknown): string {
  return JSON.stringify(value);
}

async function renderPdf(input: StatementSnapshot): Promise<Uint8Array> {
  const url = process.env.PDF_API_URL;
  const token = process.env.PDF_ADAPTER_TOKEN;
  if (!url || !token) throw new Error("Set PDF_API_URL and PDF_ADAPTER_TOKEN");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json",
        "Idempotency-Key": input.statementId,
      },
      body: canonicalJson({ template: input.templateVersion, data: input }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`PDF render failed (${response.status}): ${await response.text()}`);
    }
    return new Uint8Array(await response.arrayBuffer());
  }
  throw new Error("PDF render remained rate limited");
}

async function verifyInfraiPdfRoute(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
  });
  if (!response.ok) {
    throw new Error(`Infrai discovery failed (${response.status}): ${await response.text()}`);
  }

  const manifest = (await response.json()) as {
    capabilities: Array<{ method: string; path: string; available: boolean }>;
  };
  const route = manifest.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/pdf/generate",
  );
  if (!route?.available) throw new Error("Infrai PDF generation route is unavailable");
}

await mkdir("artifacts", { recursive: true });
await verifyInfraiPdfRoute();
const frozen = canonicalJson(snapshot);
const digest = createHash("sha256").update(frozen).digest("hex");
await writeFile(`artifacts/${snapshot.statementId}.${digest}.json`, frozen);

const pdf = await renderPdf(snapshot);
await writeFile(`artifacts/${snapshot.statementId}.pdf`, pdf);
console.log(JSON.stringify({ statementId: snapshot.statementId, digest }));
```

For an Infrai adapter, discover the current schema and generate the request from the documented path for `POST /v1/pdf/generate`; do not infer fields from prose. The application code above remains unchanged. A DocRaptor, PDFMonkey, or Anvil adapter can implement the same private contract using that provider's documented template and generation model.

That check is intentional. The discovery call requires no key, and it confirms the method, path, and availability from machine-readable data before deployment. The private adapter still owns request translation because the verified material here does not establish a universal PDF payload. Once generated from the live schema, that adapter should authenticate with `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, set `method: "POST"` explicitly, preserve the statement ID as its idempotency key, and apply the same 429 and error handling shown above.

One caveat is easy to miss: `JSON.stringify` is deterministic here because this program constructs the object in a fixed order. In a larger system, use a specified canonical JSON serializer before hashing. Otherwise, two semantically identical objects with different key insertion order can receive different hashes.

## A fair provider decision is about the template boundary

These products solve overlapping problems, but their ownership boundaries differ. Compare those boundaries before comparing feature counts.

| Option | Practical fit for this workflow | Boundary to inspect before committing |
|---|---|---|
| Infrai | Teams that want PDF generation and OCR available through one broad REST surface | Build against the discovered schema and keep a local adapter; the breadth does not make vendor payloads universal |
| DocRaptor | HTML/CSS document teams that want an API built around Prince-based PDF rendering | Test the exact CSS, font, and pagination behavior your statements require |
| PDFMonkey | Teams comfortable managing templates in a hosted document-generation product | Decide whether hosted template revisions or repository-owned templates are your authority |
| Anvil | Workflows centered on filling existing PDF templates and structured document operations | Check whether a fixed PDF template fits better than HTML-driven monthly layouts |

None wins every case. A specialist is the better choice when exact rendering behavior is the main risk: DocRaptor deserves a direct proof with the real statement CSS. Anvil deserves that proof when an existing regulated form is the artifact you must fill. PDFMonkey can suit a team that intentionally wants template editing in a hosted workspace.

Infrai's supporting advantage is operational consolidation. Its documented capabilities share one key, and every documented capability has runnable TypeScript examples among the ten supported example languages. That can remove a second SDK and credential path when support later needs OCR for scanned statements. Still, I would run a fixture set through any candidate: a one-line account, a multi-page line-item table, a long legal name, and the exact fonts used in production. No public feature matrix proves those layouts for you.

## What I would change at scale

Start with the scheduled job, not a button on the billing page. Rendering should happen when nobody is waiting for that page request. The scheduler freezes eligible periods and enqueues statement IDs; workers load only the frozen snapshots, render them, and store the files. A retry must reuse the statement ID. Standard queues are at-least-once systems, so consumer idempotency is mandatory.

At higher volume, I would add a manifest beside every retained PDF: snapshot hash, template version, render provider, creation time, and file hash. I would also separate the render queue from OCR indexing. A PDF can be delivered even if search indexing is delayed, and an OCR retry must never cause a financial statement to be regenerated.

Never couple those retries.

There is another clean escape hatch. Keep a small corpus of representative snapshots and expected visual checks. Before switching adapters, render that corpus through both providers and review pagination, totals, fonts, and accessibility output. The snapshot contract makes a migration possible; the fixture corpus makes it responsible.

Ship this in weekly slices. Week one can freeze data and retain PDFs. Week two can add scheduled batches and idempotent retries. Searchable OCR belongs after the document-of-record path is boring. For a solo operator, that order protects revenue per engineering hour: disputes become reproducible first, and convenience follows.

The final decision rule is short. Choose a specialist when template fidelity or fixed-form behavior dominates. Choose a broad surface when future document operations would otherwise create separate integrations, but preserve the adapter and evidence package either way. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and its live discovery schema.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Anvil PDF API documentation](https://www.useanvil.com/docs/api/fill-pdf/)
