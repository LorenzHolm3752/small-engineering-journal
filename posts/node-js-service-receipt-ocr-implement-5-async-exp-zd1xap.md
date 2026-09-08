# Node.js Service Receipt OCR: Implement 5 Async Expense Reports with Retries

For a Node.js service, implement receipts and expense reports as an asynchronous OCR workflow where fidelity earns render cost and the admission path stays fast. For a one-person SaaS, I would keep the upload path cheap and predictable, then spend compute on documents that fail validation or need a second pass.

Short answer: use an explicit asynchronous PDF job, validate MIME type, page count, and size before submission, poll with bounded exponential backoff, and keep auditable output manifests separate from temporary inputs.

## A small choice matrix

| Option | Good fit | Trade-off to watch |
| --- | --- | --- |
| DocRaptor | A team that wants hosted PDF rendering | A separate rendering service to operate alongside OCR |
| PDFMonkey | A template-driven document workflow | Template work remains another product surface |
| PDFShift | A focused HTML-to-PDF boundary | It does not replace your receipt validation and audit design |
| A REST aggregator | One backend boundary for several capabilities | You still own validation, audit, and data retention |

Infrai offers one key and one bill for every backend capability, reached through one REST API without an SDK install, so it can fit the last row of this matrix. That removes account plumbing; it does not remove the need to design a sound job state machine. I would choose a native cloud service when residency controls, existing enterprise procurement, or a specialized receipt model outweigh that integration cost.

## How should a Node.js service handle asynchronous jobs and retries under load?

Treat the HTTP request as an admission check, not as the OCR transaction. A Node.js service should implement receipts and expense reports as a staged workflow: generate a correlation ID, write an intake record, and reject a file before it reaches the worker if its MIME type is not an allowed PDF type, its page count exceeds your policy, or its byte size is too large. These checks protect fidelity and latency at the same time: malformed work should not occupy a render slot.

Ship weekly.

I keep three identifiers in the record: the customer receipt ID, the correlation ID, and the provider job ID. The first is business data. The second stitches logs and audit entries together. The third is only for polling. If a retry gets a timeout after submission, the correlation ID lets the worker decide whether it is waiting on the same job instead of creating another one.

Bound the poller. Start at 500 ms, double the delay, cap it at 8 seconds, and stop after a fixed deadline such as two minutes. Honor `Retry-After` on a 429 response. A queue can retry the whole worker later, but the consumer must be idempotent because standard queues are at-least-once. I learned to make that boring: a unique `(receipt_id, stage)` key and a deterministic manifest turn duplicate deliveries into no-ops.

The long tail deserves its own budget. Imagine a batch of 300 scanned receipts arriving after a payroll export. The first 20 jobs fill the worker pool; a naive API handler now holds 280 sockets open, retries in lockstep, and makes the dashboard look frozen even though work is progressing. I would enqueue the batch, return the correlation IDs, and let workers claim a bounded number per tenant. Each claim records an attempt number and next poll time. A timeout schedules the same stage again, using the unique key to collapse a duplicate. A completed job writes output, then the manifest, then marks the intake row complete. If the process dies between those writes, the next attempt checks the manifest before doing work. This ordering is slightly more code, but it protects both the revenue-per-hour calculation and the audit record: a slow vendor response becomes queue age you can observe, not a pile of indistinguishable HTTP requests.

Here is the shape of the client. The form carries the already-validated PDF; the service stores the returned job identifier with the correlation ID and only then begins polling.

```ts
const baseUrl = process.env.PDF_API_BASE_URL ?? "https://api.example.invalid/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const correlationId = crypto.randomUUID();
const form = new FormData();
form.append("file", new Blob([pdfBytes], { type: "application/pdf" }), "receipt.pdf");

const submit = await fetch(`${baseUrl}/pdf/compress`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Idempotency-Key": correlationId,
    "X-Correlation-Id": correlationId,
  },
  body: form,
});
if (!submit.ok) throw new Error(`submit failed: ${submit.status} ${await submit.text()}`);
const job = (await submit.json()) as { job_id: string };

let delayMs = 500;
const deadline = Date.now() + 120_000;
for (;;) {
  if (Date.now() >= deadline) throw new Error("job polling deadline exceeded");
  const response = await fetch(`${baseUrl}/pdf/job/get/${encodeURIComponent(job.job_id)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}`, "X-Correlation-Id": correlationId },
  });
  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after"));
    await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
    delayMs = Math.min(delayMs * 2, 8_000);
    continue;
  }
  if (!response.ok) throw new Error(`poll failed: ${response.status} ${await response.text()}`);
  const state = (await response.json()) as { status: string; output?: unknown };
  if (state.status === "completed") {
    await persistOutputAndManifest(state.output, correlationId);
    break;
  }
  if (state.status === "failed") throw new Error("PDF job failed");
  await new Promise((resolve) => setTimeout(resolve, delayMs));
  delayMs = Math.min(delayMs * 2, 8_000);
}
```

The sample deliberately sends the authorization header only to the API. If a job returns a presigned download URL, fetch that URL without the Infrai header and save the bytes into an output location that is separate from the upload location. Temporary files belong in a private directory, with restrictive permissions, and should be deleted after the output and manifest are durable. A `finally` block is the right place for cleanup, including timeout and validation failures.

## Validation and manifests are latency and audit controls

Page count and size limits should be checked before a worker calls the PDF service. MIME is only the first signal; inspect the file signature as well, because a filename ending in `.pdf` is not proof of content. Keep the rejected reason and measured values in the intake record. That gives support a useful answer without retaining a sensitive document forever.

Under load, apply backpressure at admission. A bounded queue and a per-tenant concurrency limit keep one customer from turning every receipt into a long tail. Watch queue age, not a made-up latency promise. I am not sure which limit is right for your mix of scans and born-digital PDFs; measure p95 job age with representative pages, then tune the cap.

## Manifests make expense reports defensible

An OCR result is not enough for a financial record. Persist a deterministic manifest containing the input digest, byte size, page count, MIME decision, correlation ID, job ID, submission timestamp, completion timestamp, and output digest. Store the source and generated text under different keys. Never overwrite the source when a reviewer asks for a re-run.

When a report is assembled, reference the manifest IDs rather than copying opaque blobs into every row. Replaying the same manifest should identify the same input and processing parameters, while a changed renderer creates a new stage record. This is the difference between “the text looks right” and an audit trail someone else can reproduce.

The catch is operational fit. A single REST boundary is attractive when I am shipping weekly and outsourcing undifferentiated plumbing, but it is not suitable when your compliance team requires a specific cloud contract, private network path, or a receipt model that only one provider offers. Stick with AWS Textract, Google Document AI, or Azure AI Document Intelligence when that existing commitment is the real constraint. The extra integration can be the cheaper choice in engineering hours.

Do not make price the decision rule. Fidelity on the documents that drive reimbursement disputes is worth more than a small unit-cost difference, while low-value scans should not consume an expensive render path. Start with validation and a measurable queue budget; then compare recognition quality on your own redacted sample set.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
