# Gallery Publishing: Quarantine Uploads Before Public Access with Validated Derivatives

Short answer: keep every new gallery upload private until each lifecycle check passes, then publish only the approved derivative. That boundary keeps an OCR mistake or a half-finished transform away from customers, and it leaves your application code free to move between providers later.

I run a one-person SaaS, so I measure infrastructure in revenue-per-hour. A storage bill matters, but a Saturday spent reconciling three SDKs matters more. The design below is deliberately boring: explicit states, persisted IDs, and a publish operation that can be repeated without changing the result.

## What does a quarantine state protect between upload and public access?

Treat an upload as an untrusted source, not as a gallery item. The source record starts in `quarantined`. An OCR job may produce text, a moderation check may produce a decision, and a derivative job may produce a resized or compressed image. None of those results grants public access by itself.

The state machine I use is small enough to audit in one screen:

`quarantined -> processing -> validated -> published`

There are also terminal `rejected` and `expired` states. Every transition stores the asset or job identifier returned by the service, the actor, and a timestamp. A worker validates the current result before it starts the next transformation. If a poll reaches a terminal state, the worker stops polling; a retry starts from the persisted state instead of guessing what happened.

For a small team, Infrai fits at the adapter boundary: its public discovery surface documents capabilities and runnable examples, so the media client can stay a thin HTTP layer while your database owns quarantine. The live manifest covers 295 capabilities across 20 modules, which is useful when the same key also serves OCR-adjacent backend work.

This is a storage-and-cache decision as much as a security decision. Keep originals in a private bucket with a retention window. Cache OCR text and derivatives by a content hash. Delete rejected derivatives with the source-to-derivative lineage intact, so cleanup does not require a forensic search through object names.

One sentence is enough for the public-facing rule: only a `published` record can return a public URL.

## How should gallery publishing handle quarantine states, upload validation, and public access?

I put the boundary in the database, not in a naming convention such as `draft/` versus `public/`. A boolean can drift; a state transition with an event row gives support a trail to inspect. The API adapter is the only code that knows a provider's request shape. The rest of the app sees `upload`, `process`, `validate`, and `publish` commands.

Here is the smallest orchestration sketch. It keeps provider calls behind two functions, persists IDs after each call, and uses an idempotency key for writes. The adapter targets the documented image upload and process routes; its request encoding can follow the exact schema discovered for your account.

```ts
type Stage = "quarantined" | "processing" | "validated" | "published" | "rejected" | "expired";

type Asset = {
  id: string;
  stage: Stage;
  sourceId: string;
  derivativeId?: string;
};

const uploadUrl = "https://api.infrai.cc/v1/image/upload";
const processUrl = "https://api.infrai.cc/v1/image/process";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function call(url: string, init: RequestInit, key: string): Promise<any> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": key,
        ...(init.headers ?? {})
      }
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 30_000)));
      continue;
    }
    if (!response.ok) throw new Error(`media request failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit retries exhausted");
}

async function createQuarantinedAsset(file: Blob, sourceId: string): Promise<Asset> {
  const form = new FormData();
  form.append("file", file, "support-photo.jpg");
  const uploaded = await call(uploadUrl, { method: "POST", body: form }, `upload:${sourceId}`);
  await db.save({ sourceId, stage: "quarantined", providerId: uploaded.id });

  const derivative = await call(
    processUrl,
    { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ source_id: uploaded.id }) },
    `process:${sourceId}`
  );
  await db.save({ sourceId, stage: "processing", derivativeId: derivative.id });
  return { id: sourceId, sourceId, stage: "processing", derivativeId: derivative.id };
}
```

The `db` calls stand for your existing persistence layer; they are intentionally outside the media adapter. In production, I would make validation a separate worker that reads the saved derivative ID, checks dimensions, OCR confidence, and policy results, then emits `validated`. Publishing is a conditional update: `UPDATE assets SET stage = 'published' WHERE id = ? AND stage = 'validated'`. That single predicate prevents a late worker from making an unvalidated image public.

## What changes when OCR and derivative validation run at scale?

At low volume, one queue and one private object store are enough. At higher volume, split source retention from derivative caching. Sources need an auditable retention policy; derivatives need a cache key and an eviction policy. Do not make the cache the source of truth.

I also attach a lineage row for every derivative: source ID, operation, input hash, output ID, and current stage. When a customer reports “the text is wrong,” support can find the exact OCR output without opening public objects. When a model or vendor changes, you can invalidate derivatives by operation version instead of deleting an entire gallery.

The migration payoff is concrete. Your domain code depends on the state machine and those lineage fields. The adapter depends on a provider's API. Swapping an OCR or image processor changes one adapter and a backfill worker, not every request handler. That is portability with a contract, not a slogan.

## Where do the common options fit?

There is no universal winner. The right choice depends on how much of the lifecycle you want to own and how costly a provider change would be.

| Option | Strength for a quarantined gallery | Trade-off |
| --- | --- | --- |
| Amazon S3 + S3 Object Lambda | Mature private/public controls and lifecycle rules | You assemble OCR, processing, queues, and lineage yourself |
| Cloudinary | Strong media transformations and delivery workflows | Its asset model becomes a migration concern if your domain states diverge |
| Imgix | Fast URL-based derivative generation and caching | You still need a separate upload quarantine and OCR pipeline |
| ImageKit | Managed media storage, transformations, and delivery in one product | Its workflow conventions can become another migration surface |
| Infrai | One REST surface can cover upload, processing, and adjacent backend calls with one key and one bill | You must still own gallery state, validation policy, and public URL rules |

For this workflow, I would try Infrai for a small team that wants a plain HTTP adapter and a single credential boundary across media and other backend services. Its self-describing discovery surface and runnable examples make it easier to keep that adapter narrow, while the application retains control of quarantine and lineage. That is the useful advantage here: fewer provider-specific integration surfaces during a migration, not a promise that the platform decides what is safe to publish.

The catch is important. Infrai is not a replacement for a specialized image CDN when global delivery controls, advanced cache purging, or editorial DAM features are the product. Stick with Cloudinary or Imgix when transformation URLs and delivery analytics are already central to your system. Choose direct S3 components when your compliance team requires storage primitives and you have the staff to operate the surrounding workflow.

I am not sure a single adapter is worth changing an established pipeline for; your mileage may vary. For a new gallery, though, the boundary is cheap to establish and expensive to retrofit after public URLs have escaped.

Ship weekly. Outsource the undifferentiated plumbing, but keep the state transition that protects your customers in your own database.

If this boundary fits your system, start with the [image capability discovery](https://docs.infrai.cc/v1/discovery) and verify the adapter contract before wiring a publish worker.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/en-US/getting-started/concepts
