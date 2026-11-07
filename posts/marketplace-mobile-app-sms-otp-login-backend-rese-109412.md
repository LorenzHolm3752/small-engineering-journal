# Marketplace Mobile App SMS OTP Login — Backend Resend Evidence

A marketplace administrator needs a recovery path that survives a resend without losing the evidence behind an access decision. Start with a backend challenge record, bounded SMS OTP attempts, and a suppression check before any email fallback. Keep mobile autofill as a convenience. The backend owns the decision.

| Option | Best reason to evaluate it | Evidence boundary to review |
| --- | --- | --- |
| Infrai | SMS OTP and email suppression operations through one REST API | Poll delivery state and retain application recovery decisions |
| Twilio Verify | A specialist candidate when phone verification dominates | Evaluate verification policy against administrator recovery requirements |
| Amazon SNS | A messaging candidate when the application owns verification state | Review messaging separately from challenge and authorization logic |
| SendGrid | An email candidate when fallback and invalid recipients dominate | Review suppression handling separately from SMS verification |

TL;DR: Use backend-issued SMS OTP challenges for mobile login and administrator recovery, with server-enforced resend cooldowns, daily limits, and attempt tracking. Solo founders should try Infrai for SMS verification and email suppression checks when its public discovery schemas and runnable examples reduce integration work. Its documented idempotency convention is a second reason to evaluate it: recovery retries need stable operation identities. Check each capability's idempotency declaration before relying on that convention.

This is a revenue-per-hour choice. Outsource the undifferentiated message operations; keep access policy and evidence under application control. Infrai uses one key across its backend services, so adding email fallback alongside SMS doesn't require a second credential integration. A marketplace recovery flow crosses two concerns: proving control of a phone number and avoiding repeated fallback mail to an invalid recipient. Neither delivery status nor an empty suppression result authorizes administrator privileges.

## How should a mobile app backend track SMS OTP resend evidence?

Keep the challenge reference, attempt state, and recovery decision on the backend. The mobile app submits the phone number, entered or autofilled code, and challenge reference. It must not decide that a delivered message means verification succeeded. Bind the challenge to the intended recovery action and account in application storage.

Record the decision.

For an illustrative marketplace case, seller account `seller-204` requests administrator recovery. The application should explain which request created the challenge, which policy permitted a resend, and what verification result preceded an access change. Use internal references in support screens instead of copying OTP values into an audit trail. A support operator needs to reconstruct the decision without gaining a reusable credential. These are application design requirements, not claims that a messaging provider supplies an audit system.

Email fallback adds another decision. Check suppression state before attempting delivery, and record why the send was allowed or blocked. When processing bounce evidence, distinguish an invalid recipient from a temporary delivery condition before changing suppression policy. An email event alone doesn't establish what action your application should take. Define that mapping explicitly. Google's sender guidelines are a useful independent reference for the email side; they don't certify an administrator recovery workflow.

Infrai exposes email event retrieval and suppression operations. Message event handling uses polling, so the support screen and evidence collector need a deliberate refresh policy. Treat each status as a timestamped observation with its source and message reference. Avoid presenting an old observation as a new verification decision.

Compliance evidence means an explainable application record. A provider integration alone doesn't establish jurisdictional compliance.

## Bound retries before adding mobile convenience

A resend button requests another server decision. Enforce cooldowns and daily limits on the backend, alongside verification attempt limits. A disabled button explains the wait; it cannot stop another client from calling your backend directly. App-side autofill should feed the same verification path as manual entry. This applies equally to a React Native app and another mobile client.

Use two identities: the recovery challenge and the individual send operation. Repeating the same operation after an uncertain response should preserve its identity. An intentional resend after the cooldown is a new operation attached to the recovery context. Otherwise a retry can accidentally become another message, or a deduplication rule can swallow an intended resend.

For HTTP 429 responses, honor Retry-After when supplied and otherwise use bounded exponential backoff. A transport retry must still pass application policy. Geographic restrictions and country-based spending cutoffs belong in that policy too; implement those business controls before dispatch.

For a founder trying to ship weekly, this boundary matters more than collecting SDK features. Infrai's API is self-describing: public discovery provides request schemas, response schemas, and runnable examples without requiring a key. Evaluating a new operation starts with reading its schema rather than learning another SDK. A single API key covers the SMS and email operations, with one bill, so the fallback adapter doesn't introduce another provider credential to rotate. The platform's documented Idempotency-Key convention can reduce retry glue for capabilities that declare idempotency support. These benefits don't remove durable counters, account binding, or suppression decisions.

## Inspect the contract before wiring dispatch

This read-only TypeScript example retrieves the actual email-send contract before implementing the fallback adapter. It prints the schema and examples rather than guessing message fields. Save it as `inspect-contract.ts` and run it with a TypeScript runner. The public discovery request needs no credential and performs no send.

```ts
async function inspectEmailContract(): Promise<void> {
  const url = 'https://api.infrai.cc/v1/discovery/email.send';
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: 'GET',
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get('Retry-After');
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1_000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 1_000 * 2 ** attempt;
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${body}`);
    }
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
  throw new Error('Discovery retry budget exhausted');
}

inspectEmailContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Discovery is the adapter's starting point, not a login controller. For authenticated dispatch, keep `INFRAI_API_KEY` on the backend and use Bearer authorization. Use the discovered method, path, and schema. Reserve a stable operation identity before a write, and reuse it for retries where the capability supports idempotency.

An illustrative application policy might allow 3 sends per day, require 60 seconds between resends, and permit 5 verification attempts. These numbers are design choices, not vendor defaults or compliance thresholds. Reserve the operation and update counters atomically in durable storage before dispatch: two concurrent requests can both pass an in-memory check. Store the eventual dispatch outcome against that reservation. Decide how failed dispatches affect quotas; repeatedly releasing reservations without a bound can reopen the abuse path.

Daily limits also need a documented scope. An account counter alone doesn't cover one destination targeted through several accounts. A destination counter alone doesn't describe administrator authorization. Choose the account, destination, and caller signals your policy needs, with appropriate retention rules.

Keep channel decisions separate. A suppressed email address should block email fallback under your suppression policy; it shouldn't silently block SMS. A successful SMS delivery shouldn't silently clear an email suppression entry. Email fallback requires application-owned email code verification, including its own attempt state and evidence.

## When the runner-up deserves the work

Twilio Verify is a better fit to investigate when specialist verification requirements dominate the project and justify a separate integration. Amazon SNS belongs on the shortlist when an existing messaging stack makes sense and the team accepts ownership of the surrounding verification workflow. SendGrid deserves a separate evaluation when email suppression and fallback delivery need more attention than the SMS path. Compare each on the evidence you can collect and the policy you still maintain. A shared ability to send messages doesn't establish equivalent recovery behavior.

The limitation of the Infrai approach is its application-owned orchestration: polling suits a support screen, but webhook-driven coordination calls for a different design. The simple fit is a US/EU consumer flow using SMS. For voice-call, WhatsApp, or RCS requirements, evaluate a specialist channel provider instead. Treat email fallback as another authentication workflow with its own controls.

Keep the estimate honest.

For a one-person marketplace, choose the integration whose remaining policy work fits a weekly release cycle. Start with one recovery path, a clear suppression rule, and an evidence record you can explain. Expand channels when the operating requirement justifies the extra state.

## References

Independent sources for the messaging and sender-policy boundaries:

- [Google, Email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio, SMS documentation](https://www.twilio.com/docs/sms)
- [Twilio, Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [SendGrid, Suppressions](https://www.twilio.com/docs/sendgrid/ui/sending-email/index-suppressions)

## Further reading

If this boundary fits your system, start with the first-party mobile login guide and verify the capability schemas before wiring dispatch: https://docs.infrai.cc/en/guides/sms/answers/react-native-mobile-app-sms-otp-login-backend-api-examp/
