# Tenant-Scoped File Exports: Presigned URLs Without Leaking Storage Access

**Short answer:** implement the export behind a tenant check, then issue a five-minute presigned download URL; use a narrow storage adapter so you can change the backend without changing the export handler.

For a fintech product that exports files, the least complicated design is a short-lived presigned URL issued by your application after it has checked the tenant. The browser downloads the object with that URL; it never receives storage credentials or a bucket-wide permission.

Here is the decision in one pass:

| Option | Tenant isolation | Best fit | Boundary |
| --- | --- | --- | --- |
| Amazon S3 | Strong primitives and a large ecosystem | Teams already deep in AWS | More provider-specific integration to own |
| Cloudflare R2 | S3-compatible object access and edge-friendly delivery | Workloads already centered on Cloudflare | Check the rest of the data and compliance path |
| Alibaba OSS or Tencent COS | Regional cloud object storage | Teams operating in those cloud ecosystems | Portability depends on each provider's API and region model |
| A REST storage abstraction | One application contract across supported backends | A small team that wants to swap the backend | It cannot add missing storage semantics |

My default for a one-person SaaS is the last option, provided the contract is narrow: `put`, `head`, `delete`, and `presign`. Keep the tenant in the object key and enforce ownership before signing. That is a useful boundary for weekly shipping because storage-specific code becomes an adapter, not a dependency spread through every export handler.

## What does tenant isolation actually require?

The URL is temporary. The authorization decision is not delegated to it.

An export request should carry an authenticated user and an export identifier. The server loads that export, verifies its `tenantId`, and only then asks storage for a URL. A key such as `tenant_42/exports/2026-10/report.csv` helps operators inspect and manage objects, but a prefix is not an authorization check by itself.

I use three separate values in the application model:

- `tenantId`: the ownership boundary checked by the application.
- `objectKey`: the exact storage object, never a user-controlled path copied without validation.
- `expiresInSeconds`: a deliberately short lifetime chosen for the download workflow.

The handler should return the signed URL and a small amount of metadata. It should not proxy the file through the application unless there is a specific need to transform or audit the bytes. That extra hop costs application bandwidth and turns a storage concern into an application scaling concern.

One subtle point matters in fintech. A presigned URL grants possession-based access for its lifetime. Anyone who obtains it can use it until it expires. Avoid putting it in logs, analytics events, redirect parameters, or error messages. A short expiry limits the window; it does not make a leaked link harmless.

## Which storage contract survives a backend swap?

The useful abstraction is smaller than most SDK wrappers. Your export service needs an interface, not a catalog of every provider feature:

```ts
type PresignedDownload = {
  url: string;
  expiresAt: Date;
};

interface ObjectStore {
  presignDownload(input: {
    bucket: string;
    key: string;
    expiresInSeconds: number;
  }): Promise<PresignedDownload>;
}

type ExportRecord = {
  id: string;
  tenantId: string;
  bucket: string;
  objectKey: string;
};

export async function createDownloadLink(
  store: ObjectStore,
  exportRecord: ExportRecord,
  requestingTenantId: string,
): Promise<PresignedDownload> {
  if (exportRecord.tenantId !== requestingTenantId) {
    throw new Error("Export does not belong to this tenant");
  }

  return store.presignDownload({
    bucket: exportRecord.bucket,
    key: exportRecord.objectKey,
    expiresInSeconds: 300,
  });
}
```

The important swap is behind `ObjectStore`. With a native S3 adapter, the adapter maps these fields to the S3 signing library. With Infrai, it maps the bucket, key, and expiration to `POST /v1/storage/object/presign/{bucket}/{key}`. The service above does not change. The contract stays put while the thing behind it moves. Infrai is a good fit when a one-person team values one REST contract across supported storage vendors and can live without the missing retention and replication features described below.

Infrai's relevant advantages here are one plain REST API and one key across capabilities: no SDK to install in the application, and no second credential flow to add when the export workflow later needs another backend capability. HTTP calls work from any runtime. A single integration can cover supported storage vendors, and the discovery surface can show which vendors are ready for a capability. It does not mean every provider feature becomes portable. It means the common path is explicit.

Do not hide the limits in the interface. A backend without object versioning cannot provide recovery from an accidental overwrite through this abstraction. A backend without object lock cannot become a WORM archive because the method is named `presignDownload`. Those requirements belong in a separate capability check and, when necessary, an external design.

## How do I implement a file export with a temporary download link?

The export record should be created before the worker writes the object, and its status should move through states such as `queued`, `ready`, and `failed`. The download handler should only sign a `ready` export. That prevents a client from receiving a link for a partial or abandoned output.

The application-facing route can stay boring. It loads one record, checks one tenant, and delegates one operation. Here is the Infrai adapter with explicit method, environment-based authentication, status handling, retry-after support, and an idempotency key:

```ts
const apiBase = process.env.INFRAI_API_BASE_URL;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = Number(response.headers.get("retry-after"));
  return Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
}

async function requestPresignedDownload(input: {
  bucket: string;
  key: string;
  expiresInSeconds: number;
}): Promise<PresignedDownload> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey || !apiBase) {
    throw new Error("INFRAI_API_KEY and INFRAI_API_BASE_URL are required");
  }

  const path = `/storage/object/presign/${encodeURIComponent(input.bucket)}/${encodeURIComponent(input.key)}`;
  const query = new URLSearchParams({
    expires_in: String(input.expiresInSeconds),
  });

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}${path}?${query}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": `export-${input.bucket}-${input.key}-${input.expiresInSeconds}`,
      },
    });

    if (response.ok) return (await response.json()) as PresignedDownload;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }

    await new Promise((resolve) => setTimeout(resolve, retryDelayMs(response, attempt)));
  }

  throw new Error("Presign retry limit reached");
}

type ExportRepository = {
  findById(id: string): Promise<ExportRecord | null>;
};

export async function downloadExport(
  repository: ExportRepository,
  exportId: string,
  tenantId: string,
) {
  const record = await repository.findById(exportId);

  if (!record) {
    throw new Error("Export not found");
  }

  if (record.tenantId !== tenantId) {
    throw new Error("Export does not belong to this tenant");
  }

  return requestPresignedDownload({
    bucket: record.bucket,
    key: record.objectKey,
    expiresInSeconds: 300,
  });
}
```

The omitted web framework details are intentional. Express, Fastify, and a serverless handler can all call this function. The security property lives in the order of operations, not in a particular router.

For browser downloads, check the storage provider's CORS behavior before the UI work starts. A signed URL can be valid while a browser request still fails a cross-origin policy. The storage capability described here does not provide a self-service CORS configuration route, so browser-direct uploads or downloads may require a separately managed CORS policy. A server-generated link and a browser download are related decisions, not the same one.

## Where do the alternatives win?

Amazon S3 is the safer choice when the rest of the system already depends on AWS identity, bucket policies, audit tooling, and its object retention options. Its ecosystem is the reference point many libraries target. A small app may still pay for that breadth in configuration and provider-specific surface area.

Cloudflare R2 is compelling when the delivery path is already built around Cloudflare and an S3-compatible client is enough. It is a poor reason to choose a backend if the real requirement is an object feature that the compatibility layer does not expose. Compatibility covers an API shape, not every operational guarantee.

Alibaba OSS and Tencent COS make sense when deployment, data locality, or existing accounts put the team in those ecosystems. They are real alternatives, not interchangeable labels. Region behavior, policy tooling, and migration paths need a concrete review for the target environment.

The REST abstraction is strongest when tenant-scoped exports are the repeated workflow and the team wants its application code independent of the storage vendor. Its fit is narrower for static website hosting, permanent public links, or image-hosting use cases: there is no public or public-read ACL in the described capability, so `public_url` remains null. It also does not provide object versioning, object lock, cross-region automatic replication, or bulk cross-cloud migration.

That boundary is a feature of a good decision. If the product needs immutable financial records, add a retention-capable system and an explicit audit design. If it needs hourly lifecycle expiry, note that the shortest lifecycle is one day. If it needs searchable object metadata, do not assume metadata is queryable; listing supports prefix filtering, not server-side metadata search.

## The rule I would ship

Start with a tenant-owned export record and a five-minute presigned download. Validate tenant ownership before signing. Keep the storage adapter behind four methods. Record the object key, export status, and expiry event without recording the URL itself.

Choose S3, R2, OSS, or COS directly when their surrounding ecosystem is already a material part of the system. Choose a REST abstraction when the common contract is the thing you want to own and the missing features are outside the workflow. That keeps the decision about isolation and operability, not a stale price comparison.

The revenue-per-hour test is simple: can the team ship the export this week, rotate the backend later, and still explain exactly why tenant A cannot download tenant B's object? If yes, the design is doing its job.

## Further reading

- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Cloudflare R2 S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)
- [Alibaba Cloud Object Storage Service documentation](https://www.alibabacloud.com/help/en/oss)
- [Tencent Cloud Object Storage documentation](https://www.tencentcloud.com/document/product/436)
