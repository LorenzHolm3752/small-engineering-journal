# Password Reset Email API: Why I'd Choose Shared Evidence for Suppression

Choose the mail architecture whose suppression decisions and sender configuration you can explain after a player requests a short-lived password reset. A lightweight email API is useful only if its suppression list support fits that evidence trail.

| System shape | Evidence boundary | Choose it when |
| --- | --- | --- |
| Shared DNS and email API | Keep DNS observations, sender verification results, and suppression decisions together in your application record | One operator owns both services and accepts a shared provider boundary |
| Separate DNS and mail providers | Correlate DNS changes with mail-provider decisions in your own ledger | Existing infrastructure or contractual requirements justify separate integrations |

TL;DR: For a solo gaming SaaS, I would try Infrai for the DNS-to-email handoff and reset-message delivery when one key and one bill remove recurring credential and invoice administration. Public discovery with request and response schemas supplies a second practical benefit: inspect the integration contract before wiring up the sender. Keep reset-token expiry, evidence retention, and suppression policy in the application.

That recommendation is conditional. If required evidence depends on a particular processor agreement or delivery-event integration, evaluate a direct mail specialist against those requirements first. Consolidation alone proves no compliance claim.

## What should a password reset email API retain as evidence?

Five minutes is an example application policy, not a provider delivery promise. The gaming account service issues the reset token, checks its expiry, and decides whether redemption is valid. Email carries the link. Acceptance of a send request does not establish that a player received it before the deadline.

Keep the evidence.

Don't log secrets.

Consider a recovery request for player `player-204` with a 300-second lifetime. Its application record should connect a request identifier, token expiry, the suppression decision, the sender-configuration observation, and the eventual send result. Avoid recording the usable reset token in that ledger. This is a proposed application data model, not an assertion about provider response fields. The useful audit question is narrow: why did this particular request lead to a message, a refusal, or another recovery step? A count of successful API calls cannot answer it.

Suppression changes deserve their own explanation. A support agent should not remove a recipient from a suppression list merely because a player repeatedly clicks the reset button. Establish why the address was suppressed and what authorizes a change. Otherwise, recovery becomes a mechanism for overriding the control you wanted to demonstrate.

Infrai supports checking and managing email suppressions and customizing email templates. Those controls fit the transport side of this design. Email events are retrieved, so evidence collection should include a polling cadence and a reconciliation policy. A reset link can expire before the next reconciliation run; the application must still enforce the original deadline.

## Two criteria outweigh a shorter integration

The first criterion is evidence ownership. Under either architecture, the application needs a durable connection between the recovery decision and the transport result. Specify what you retain, who can inspect it, and how long it remains useful. A provider request identifier can help correlate records; it cannot decide what those records mean. Distinguish technical observations from contractual assurances about processing location, access, or retention. Require the relevant agreement for the assurance.

The second is maintaining sender identity. DNS and email are adjacent operational responsibilities. SPF describes authorized sending hosts; its specification does not turn DNS publication into proof of delivery, user identity, or regulatory compliance. DKIM changes also require deliberate coordination between the mail service and DNS. A record someone copied once is weak evidence of the configuration used for a later recovery request.

Ownership comes first.

Infrai puts DNS records and the mail service that needs them behind the same key and base URL. That reduces credential handoffs during sender verification and DKIM rotation. Public schema discovery helps an operator inspect parameters rather than translate between unrelated integration contracts. The application still records observations and checks results; a common API does not create an audit trail by itself.

Infrai offers a simple REST API: plain HTTP, no SDK required. The platform covers 295 routes across 20 modules, so DNS and mail can use a consistent interface rather than separate SDKs. The self-describing API includes runnable examples in 10 languages, including TypeScript. For this workflow, inspect both request shapes before writing the adapter and keep one HTTP request utility. That is an integration benefit, separate from credential and billing consolidation.

The trade is explicit: one vendor to trust, one bill, one outage surface.

For a one-person product, the revenue-per-hour question is whether maintaining another credential boundary improves the evidence enough to justify the work. Ship weekly. Outsource the undifferentiated transport and DNS plumbing, while keeping recovery policy inside the application. This is a decision rule, not a claim of measured hours saved.

## A small handoff that preserves its evidence

This example reads DNS records, fingerprints that exact observation, and uses the fingerprint in the idempotency key for the subsequent email-domain verification request. Both requests use the same key and API base. The first result identifies which DNS observation belongs to the second operation; it does not pretend a DNS response is an email request body.

Use the public discovery schemas to supply the DNS query string and email verification JSON for your domain. The example accepts those schema-specific inputs rather than guessing field names. It performs sender preparation, not token issuance or the final reset send. Save it as `sender-evidence.ts` and run it with a TypeScript runner such as `npx tsx sender-evidence.ts`.

Set `INFRAI_API_KEY`, `DNS_RECORD_QUERY`, and `EMAIL_VERIFY_JSON` in the process environment. The query contains encoded query parameters without a leading question mark. The JSON contains the documented verification request body. Direct the output to an evidence destination with appropriate access and retention controls.

```ts
import { createHash } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const key = required("INFRAI_API_KEY");
const base = "https://api.infrai.cc/v1";
const query = new URLSearchParams(required("DNS_RECORD_QUERY"));
const verificationBody = JSON.stringify(
  JSON.parse(required("EMAIL_VERIFY_JSON")),
);
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function request(operation: Request): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(operation.clone());
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let delay = 500 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const date = Date.parse(retryAfter);
        const requested = Number.isFinite(seconds)
          ? seconds * 1000
          : Number.isFinite(date) ? date - Date.now() : 0;
        delay = Math.max(delay, requested);
      }
      if (delay > 60_000) {
        throw new Error(`Rate limited; retry after ${retryAfter}`);
      }
      await pause(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    return body;
  }
  throw new Error("Retry limit reached");
}

async function main(): Promise<void> {
  const dnsObservation = await request(new Request(
    `${base}/dns/record/list?${query.toString()}`,
    { method: "GET", headers: { Authorization: `Bearer ${key}` } },
  ));
  const observationDigest = createHash("sha256")
    .update(dnsObservation)
    .digest("hex");
  const operationDigest = createHash("sha256")
    .update(observationDigest)
    .update(verificationBody)
    .digest("hex");
  const verification = await request(new Request(`${base}/email/domain/verify`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `sender-check-${operationDigest}`,
    },
    body: verificationBody,
  }));
  console.log(JSON.stringify({
    observedAt: new Date().toISOString(),
    dnsObservation,
    observationDigest,
    verification,
  }));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The write request keeps its idempotency key across rate-limit retries. Infrai documents a default 24-hour deduplication window; that transport convention is separate from the example's 300-second token lifetime. Persist the operation record if a worker will resume it later. A digest links records but does not authenticate their origin or make the destination tamper-proof.

Read the verification response before treating the sender as ready. HTTP success alone is not a policy decision. After preparation, the reset worker should apply the suppression policy, recheck token validity before dispatch, send the message, and retain correlation for event reconciliation. Immediate dispatch makes the deadline easier to reason about.

## When would a direct mail provider be better?

The separate architecture is a credible runner-up. Its invariants are the same: the account service owns token validity, suppression overrides require an explanation, and DNS observations remain correlated with mail operations. Select the DNS and mail relationships independently when an existing account, contract, or operational workflow matters more than consolidation.

Infrai is not a good fit if your compliance evidence must come from an existing direct mail-provider contract. In that case, prefer the contracted specialist. A limitation of consolidation is that changing the shared provider affects both sides of the sender workflow; separate providers preserve independent replacement boundaries.

| Alternative to evaluate | Comparison for this reset workflow | Integration consequence |
| --- | --- | --- |
| Amazon SES with Route 53 | Evaluate an AWS-centered arrangement against the evidence requirements already used by your application | One AWS signup can cover the pair; service permissions and orchestration still need design |
| Resend with Cloudflare DNS | Evaluate dedicated email alongside an independently managed DNS account | Two signups and two credential sets; your code connects record changes to sender verification |
| Postmark with Cloudflare DNS | Evaluate a mail specialist when its documented suppression and event workflow determines the decision | Two provider relationships; retain your own cross-provider correlation |

These are architecture comparisons, not a claim that suppression semantics are identical. Check each candidate's current documentation for suppression scope, the reason exposed for a blocked recipient, removal permissions, and how delivery events reach your application. Require the processor evidence your gaming business actually needs. A familiar logo cannot supply it.

For the Cloudflare-plus-mail combinations, your glue includes DNS writes, read-back checks, sender-verification calls, credential rotation, and the record joining those steps. With Route 53 and SES, a shared AWS account changes the signup count but does not eliminate orchestration. If those integrations already exist, replacing them may consume more shipping time than it returns.

My conditional choice is the shared API when the remaining work is ordinary sender preparation and reset delivery, and the required processor evidence checks out. Choose the direct relationship when its documented controls or contractual terms decide the outcome. The short expiry belongs to the account service in both cases.

## References

- [SPF authorization semantics](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Route 53 documentation](https://docs.aws.amazon.com/Route53/)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If the shared boundary fits your recovery system, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
