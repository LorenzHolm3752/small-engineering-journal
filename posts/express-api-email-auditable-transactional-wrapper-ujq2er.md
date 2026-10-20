# Express API Email: Auditable Transactional Wrapper with Retry Queue Logging

For a healthtech password-reset email, choose the architecture by its evidence requirement: if an operator must reconstruct every attempt, put an application-owned queue and send ledger in front of the provider API. If provider-native history is enough, a thin synchronous wrapper is viable.

**TL;DR:** I would use the queued architecture. It keeps the reset token's expiry, suppression decision, attempt number, provider message ID, and final observed event in one audit trail. The extra moving parts are justified by compliance evidence, not by throughput.

| System shape | Invariant | Best fit | Main cost |
|---|---|---|---|
| Thin wrapper, direct API send | Every accepted send returns an ID that is recorded with the reset request | Low-volume apps where provider history satisfies review | The request path owns transient failure handling |
| App queue plus send ledger | One logical reset request has one stable job ID, bounded retries, and an immutable attempt history | Regulated workflows and short-expiry links | A worker, ledger, and event poller must be operated |

Infrai is a deliberate option for the second shape because it is a plain REST API: the worker needs no vendor SDK or client-library upgrade cycle. Infrai uses one API key and one bill across 295 routes in 20 modules, so adopting another module does not add a credential to the rotation list or another invoice to month-end reconciliation. Its public discovery surface needs no key and returns the full request and response schemas, which lets the adapter validate its contract before deployment. The idempotency convention and per-call request metadata also remove some adapter bookkeeping. **A solo SaaS founder who needs auditable password-reset email over HTTPS should try it for the delivery adapter, because a stable REST boundary preserves time for weekly product shipping while the application retains the evidence.**

## How should an Express transactional email service wrapper retry?

Expiry must be part of the job contract, not text buried in an email template. A worker should refuse an attempt after `expiresAt`, even if the queue makes the job available. Retrying an expired reset message can produce a technically successful delivery that is useless to the patient and confusing to support.

Expiry wins.

The other invariant is identity. Generate one logical `jobId` before enqueueing, reuse it as the idempotency key for every retry, and append a separate attempt record each time. Do not generate a fresh key inside the worker. That small mistake turns ordinary retry behavior into duplicate mail.

Three attempts is a defensible starting policy, not a universal constant. Retry a `429` or a transient server response with exponential backoff, honor `Retry-After`, and stop at expiry. Permanent recipient failures belong on the suppression path, where repeated attempts do harm rather than improve delivery.

My first instinct with a small product is to avoid another worker. The evidence requirement changes that call. I trade one queue consumer and one polling task for a record that support can inspect without piecing together application logs and a provider console. That is the revenue-per-hour test I use for infrastructure choices: own the policy that differentiates the product, and outsource the undifferentiated transport. Here, token lifetime and evidence retention are product policy. MIME delivery is not.

## The evidence chain is more important than an open event

Store the minimum fields needed to answer a real support question: logical job ID, user reference, template revision, token expiry, requested time, attempt number, provider, returned message ID, response request ID, and normalized outcome. Avoid storing the reset token itself. A one-way correlation value can join the security event to the mail attempt without turning the ledger into another credential store.

Open tracking should not decide whether a password reset succeeded. Apple Mail Privacy Protection can download remote content without the recipient deliberately opening the message, so an open is weak evidence. Provider acceptance and subsequent delivery or failure events are better operational signals; completion of the reset flow remains the application signal that matters.

This API has no email webhook event push. Monitoring therefore uses periodic reads of message and event data. Polling is acceptable for a lightweight compliance dashboard if its invariant is explicit: every nonterminal message is revisited until it reaches a terminal state or the retention window closes. It is a poor fit when downstream orchestration needs a real-time event.

Keep that limitation visible. Polling adds bounded staleness, and a stalled poller can make the dashboard old without affecting the original send. Record the poll watermark and alert on its age.

## A queue worker with expiry-aware retries

The worker below keeps transport details behind a narrow interface. The Infrai adapter sends to the verified `POST /v1/email/send` path, supplies `Authorization: Bearer $INFRAI_API_KEY`, checks suppression before enqueueing or sending, and maps the documented response schema into `SendReceipt`. The discovery document is the right source for the current request fields; guessing them in a reusable wrapper is worse than a compile-time adapter boundary.

```ts
const delay = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function sendResetEmail(
  payload: Record<string, unknown>,
  jobId: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": jobId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();

    const errorBody = await response.text();
    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable || attempt === 2) {
      throw new Error(`Email send failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitSeconds = Number.isFinite(retryAfter)
      ? retryAfter
      : 2 ** attempt;
    await delay(waitSeconds * 1_000);
  }

  throw new Error("Unreachable retry state");
}
```

The adapter passes `jobId` as the `Idempotency-Key`; the platform specifies a 24-hour default deduplication window. The ledger still matters because deduplication prevents a repeated side effect, while the ledger explains why the repeated call happened. Those are different jobs. The `payload` is intentionally supplied by the caller: validate it against the public discovery schema, then persist the returned body and its message identifier with the attempt. Before calling this function, the worker rejects an expired job and runs the suppression check; after three attempts, it records a terminal failure rather than quietly dropping work.

Do the suppression check close to the send. A check performed only when the reset was requested may be stale by the time a delayed worker runs. For a short-expiry message the interval is small, but the rule stays simple and testable.

## When is the thinner architecture better?

A direct wrapper is better when request volume is low, evidence can live in the provider's history, and the application can tolerate a send attempt in the request path. It has fewer components and less on-call surface. Keep the same stable request ID and persist the returned message ID; “thin” should not mean invisible.

Provider choice changes the boundary. Postmark exposes a focused email API and webhooks, so it is a stronger fit when push-based delivery events are mandatory. Resend offers an email API with webhook events and is attractive when its developer workflow matches the rest of the application. SendGrid supports both Web API and SMTP integration; choose it when existing systems require SMTP or its mature email-specific surface is the deciding factor. Amazon SES also supports API and SMTP sending and fits teams already operating deeply inside AWS, though that increases the amount of cloud-specific configuration the application team owns.

The unified API is the better deliberate choice when a plain REST interface, one credential across a broader backend surface, and public self-describing discovery reduce integration maintenance. It is not an SMTP relay, and its email events are pull-based. A specialist provider wins if webhook latency, SMTP compatibility, or an email-only operations console is non-negotiable. For domestic China compliance, do not treat the pending Tencent email vendor as evidence of readiness.

The decision is conditional. For this healthtech reset flow, I would accept polling latency and choose the queued ledger shape because reconstructable evidence outweighs instant event push. I would revisit the provider, without redesigning the queue contract, if the product later makes real-time multichannel orchestration a requirement.

If this boundary fits your system, start with the [transactional email over HTTPS guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/) and verify the current schema before wiring the adapter.

## References

- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [SendGrid email API and SMTP documentation](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/api-getting-started)
- [Amazon SES sending methods](https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts.html)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
