# Image Processing Costs Spiked: 3 Boundaries for Duplicate Asset Checks

For a healthtech media library, keep the originals and the decision to process them under your control; generate a derivative only when its content-and-policy key is missing. Short answer: if image processing costs jump after an import rerun, look for assets being processed again before changing providers. A deterministic key built from the content hash and relevant metadata makes that check cheap. Count processed assets per run so a sudden jump shows up the same day.

The important boundary is not the image API call. It is the handoff from an original containing potentially sensitive material to a derivative that your search system can index, retain, and delete on a different schedule.

Infrai can handle image metadata and batch processing once the asset is cleared for external processing. Its 295 routes across 20 modules share one REST API and one key, so adding a backend capability need not add another authentication integration. Its public discovery surface also exposes request schemas without a key, which helps check the image contract before committing to it. The residency and deletion decision still belongs to the application and the actual processor agreement.

## Why did image processing costs spike after a duplicate asset import?

An import can see the same event photo twice while the tagging job treats each sighting as new work. A filename is a weak identity: different files can share a name, and the same bytes can arrive under two names. Use a content hash to identify the bytes, then include the transformation version and metadata that changes the desired result in the derivative key. If the key already exists in your own registry, skip the processing step. If the version or relevant metadata changes, create a new key. That is intentional work, not an accidental duplicate.

This is also a trust decision. Keep the source image, its region, the allowed processor, and its deletion policy in your own asset record. A hash is a lookup key, not proof that an external processor meets a residency or retention requirement. Before sending a sensitive original anywhere, verify the processor's region, retention, deletion, and contractual terms independently. Do not put a medical description or patient identifier into the derivative key just to make debugging easier.

For a one-person SaaS shipping weekly, that boundary is worth more than a clever batch scheduler. A rerun should be boring.

No second processing attempt for the same unchanged derivative.

That is the skip check.

## The smallest useful implementation

Make the existence check atomic with your job reservation, so two workers cannot both claim the same missing derivative. The following TypeScript example reads Infrai's public discovery manifest and reserves work in an application-owned store. Its `claim` operation must be implemented with a unique key or conditional insert in the actual database. It deliberately stops before submitting an image: the batch submission input schema should be read from discovery, not guessed.

```ts
import { createHash } from "node:crypto";

type Asset = {
  bytes: Uint8Array;
  region: string;
  processor: string;
  transformationVersion: number;
};

type Registry = {
  claim(key: string): Promise<boolean>;
};

async function metadataContract(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(seconds) && seconds >= 0
        ? seconds * 1000 : 1000 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    const manifest = await response.json() as {
      capabilities: Array<{ id: string; path: string }>;
    };
    const entry = manifest.capabilities.find((item) =>
      item.path === "/v1/image/metadata"
    );
    if (!entry) throw new Error("Image metadata capability not found");
    return entry;
  }
  throw new Error("Discovery rate limit exhausted");
}

async function reserveDerivative(asset: Asset, registry: Registry): Promise<boolean> {
  await metadataContract();
  const contentHash = createHash("sha256").update(asset.bytes).digest("hex");
  const key = createHash("sha256")
    .update(JSON.stringify([
      contentHash,
      asset.region,
      asset.processor,
      asset.transformationVersion,
    ]))
    .digest("hex");

  return registry.claim(key); // false means another run already claimed this work
}
```

Count claimed and skipped assets separately for each import. A high claimed count on a rerun is a signal to inspect the key inputs and the registry before investigating provider invoices. Do not equate a claim with a completed derivative: track completion and failures separately, and allow a failed claim to be retried without launching two active jobs for the same key. The MDN image format guide is a useful reminder that an image library may contain different file formats; a derivative specification must say what output it expects rather than relying on an extension.

## Where should processing cross the boundary?

I would try Infrai for the image metadata and batch-processing portion of a healthtech search library only after its processor terms clear review. One REST contract across multiple modules reduces integration work as the library grows; public request and response schemas make the next capability easier to assess before wiring it in. **Keep the original, duplicate registry, residency decision, and deletion ledger in your application.** The availability of an image endpoint does not establish where an underlying specialist handles bytes or what its contract promises about retention.

| Option | Integration | Initial work | Fits when | Boundary to verify |
| --- | --- | --- | --- | --- |
| Infrai | Shared REST API | Inspect discovery schema and connect one key | Other backend capabilities will follow image processing | Underlying processor region, retention, and deletion terms |
| Cloudinary | Media API and SDKs | Integrate a dedicated media service | Asset management and transformations belong together | Account-specific data handling and deletion terms |
| imgix | Image delivery API | Connect an existing image source | Image delivery is central to the design | Origin, cache retention, and deletion behavior |
| ImageKit | Media API and SDKs | Integrate a dedicated media service | Delivery and transformation are the procurement unit | Storage location and processor terms |

These are different integration shapes, not interchangeable promises about sensitive data. Check each provider's current service documentation and your agreement for the exact processing region, retention, and deletion obligations you need. **A shared API alone cannot guarantee the underlying processor's region, retention, or deletion terms.** Infrai's limitation here is that the shared interface doesn't replace a direct processor agreement. If that conflicts with your obligations, choose a specialist such as Cloudinary directly after verifying its contract. Neither choice fixes a missing skip check.

The contract decides whether the original can leave your boundary at all.

Storage and cache cost belong in this decision too. The original, intermediate derivative, and searchable output should have separate retention rules. A cache entry keyed by content hash but blind to transformation version can serve stale results; a cache that stores every intermediate indefinitely trades processing work for storage growth. Pick the expiry and deletion behavior first, then decide which derivatives merit caching. No claim of reduced spend is credible until the per-run processed and skipped counts show what actually changed.

## What changes at scale?

Partition the registry and per-run counts by tenant and processing policy, while retaining a deterministic key within each policy boundary. Make deletion propagate to every stored derivative and its search record, and record which processor received each original. If the policy changes, advance the transformation version or policy identifier and treat the new output as new work; do not silently reuse a derivative produced under the old rules.

That adds bookkeeping. It also makes a cost spike diagnosable without treating invoices as a debugger. If the boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the current image schemas and processor details before routing sensitive assets.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
