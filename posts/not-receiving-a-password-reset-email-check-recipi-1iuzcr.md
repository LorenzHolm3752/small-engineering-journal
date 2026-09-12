# Not Receiving a Password Reset Email? Check Recipient Suppression Before Removal

| Evidence available | First place to look | Do not do yet |
| --- | --- | --- |
| No internal reset job | Application and queue boundary | Change mail settings |
| A job, but no transport handoff | Worker execution | Delete a recipient suppression |
| Handoff plus a bounce | Address history and suppression record | Send the same message again |
| Handoff with no recipient event | Event ingestion and message trace | Assume delivery |
| Failures across many domains | Sending identity and SPF record | Treat it as one bad mailbox |

Short answer: when a recipient is not receiving a password reset email, trace one request until the first missing state, check the suppression record only after you have bounce evidence, and remove that record only when the address is valid and the reason that created it has been resolved. Then create a fresh, single-use reset token. A successful web response alone proves none of those delivery steps.

That is the smallest useful decision. It protects the account while keeping a support ticket from turning into an open-ended deliverability project. For a one-person SaaS, that distinction matters: revenue per hour favors a narrow trace that can ship this week, while security favors refusing a quick suppression-list deletion when the evidence is weak.

## How should you check a suppression list when a password reset email bounced?

Start with a correlation ID from the reset request, then follow it through the queue and mail handoff. The goal isn't to collect every log line. It is to find the first transition for which you have no evidence.

Use a state chain such as `requested -> queued -> handed_off -> recipient_event -> reset_completed`. Keep reset completion separate from mail delivery. Store the transport's message identifier alongside your internal job identifier, but never put a reset token in logs. A keyed representation of the address can make internal searches possible without spreading the full address through every operational system.

This ordering prevents three common category errors. If the request never created a job, recipient suppression is irrelevant. If a worker never handed the message to the transport, DNS is not the first problem to solve. If the handoff exists and the exact address has a bounce-linked suppression, then the address-level history deserves attention.

No handoff, no mail trace.

A suppression entry should be treated as a decision input, not a delete button. Classify why it exists. An invalid or retired mailbox should remain blocked. A complaint or an unsubscribe reflects recipient intent and should not be erased merely because someone opened a support ticket. A bounce-related entry can be reviewed, but removal should require evidence that the mailbox is now valid, that the account owner requested another reset, and that the condition behind the bounce has changed. After an approved removal, issue a new reset request rather than replaying an old message. OWASP's Forgot Password Cheat Sheet calls for reset tokens that are random, stored securely, long enough to resist brute force, linked to one user, invalidated after use, and expired after an appropriate period. It also recommends a side channel for delivery and says an account should not change until a valid token is presented. Those properties make an old link the wrong recovery artifact even if it happens to be unexpired. Keep the public endpoint boring, too: OWASP recommends the same response for existing and nonexistent accounts, response timing that is consistent enough to avoid an account-enumeration signal, and protection against excessive automated requests. The browser can receive a generic acknowledgement while the internal trace records the states operators need. Don't expose suppression status, account existence, or the transport response to the requester.

## Two criteria decide where the investigation goes

The first criterion is scope. One address with a recorded bounce points toward recipient history. A group of failures that starts at the same deployment boundary points toward application or worker behavior. Failures spread across addresses and domains call for a wider review of handoff, event intake, and sending identity. These are diagnostic branches, not claims that any one signal proves the final cause.

SPF belongs on the broad branch. RFC 7208 defines SPF as a way for a domain to explicitly authorize hosts to use its names in the `HELO` and `MAIL FROM` identities during SMTP transactions. That makes an SPF check relevant to authorization of the sending identity. It does not tell you that a particular reset message reached an inbox, and it does not explain why one exact address appears in a suppression store. Keep those questions separate.

The second criterion is reversibility. Looking up an event or a suppression record is read-only. Removing a suppression changes future sending behavior. Generating another token creates another credential. A practical support flow should move from low-risk inspection to higher-risk mutation, requiring better evidence at each step.

This is where the catch lives: a manual review is not suitable when legitimate address corrections arrive at high volume. The queue will become slow and inconsistent. Use a role-controlled review workflow with an explicit policy when volume demands it, and keep complaints and unsubscribes outside automatic removal. At the other extreme, an append-only delivery ledger can be needless machinery for a tiny service with rare recovery tickets. Stick with correlation IDs, a durable queue, searchable message events, and an audited manual decision while a case can still be traced quickly.

I'm not sure a universal bounce-rate threshold would improve this workflow. Audience, sending pattern, and recipient mix vary, and neither cited source supplies such a threshold. What can be tested is the state machine: every accepted internal job should either advance to its next recorded state or become visible for investigation within the service's own chosen window.

## A small TypeScript boundary keeps the policy testable

Outsource the undifferentiated transport. Keep the removal decision in application code you can test and audit. The interface below deliberately avoids vendor routes and event names; an adapter translates the external system into the few states the recovery policy needs.

```ts
type SuppressionKind = "bounce" | "complaint" | "unsubscribe";

type SuppressionRecord = {
  kind: SuppressionKind;
  recordedAt: Date;
};

type RemovalReview = {
  mailboxIsValid: boolean;
  ownerRequestedNewReset: boolean;
  bounceCauseResolved: boolean;
};

interface RecoveryMailPort {
  findSuppression(recipientId: string): Promise<SuppressionRecord | null>;
  removeBounceSuppression(recipientId: string): Promise<void>;
  createFreshReset(recipientId: string): Promise<void>;
  recordDecision(recipientId: string, outcome: ReviewOutcome): Promise<void>;
}

type ReviewOutcome =
  | "no-suppression"
  | "kept"
  | "bounce-removed-and-reset-created";

async function reviewResetDelivery(
  mail: RecoveryMailPort,
  recipientId: string,
  review: RemovalReview,
): Promise<ReviewOutcome> {
  const record = await mail.findSuppression(recipientId);

  if (!record) {
    await mail.recordDecision(recipientId, "no-suppression");
    return "no-suppression";
  }

  const removalApproved =
    record.kind === "bounce" &&
    review.mailboxIsValid &&
    review.ownerRequestedNewReset &&
    review.bounceCauseResolved;

  if (!removalApproved) {
    await mail.recordDecision(recipientId, "kept");
    return "kept";
  }

  await mail.removeBounceSuppression(recipientId);
  await mail.createFreshReset(recipientId);
  await mail.recordDecision(recipientId, "bounce-removed-and-reset-created");
  return "bounce-removed-and-reset-created";
}
```

The important branch is the refusal path. Complaints and unsubscribes cannot reach the removal operation. A bounce still needs three approvals, and the reset is created only after that review passes. In production, `recipientId` should resolve to the address inside a restricted adapter, while the audit record captures the operator, correlation ID, reason, timestamp, and outcome without capturing the token.

Test the policy as a table, not as one happy-path assertion. Cover no record, each non-bounce kind, each missing approval, and the fully approved bounce case. At the integration boundary, verify that duplicate external events don't produce duplicate state transitions and that a repeated internal job cannot create an uncontrolled stream of reset messages. The exact idempotency mechanism belongs to the application and queue design, so choose it explicitly rather than assuming the mail transport provides it.

Ship the narrow version first.

## When is the runner-up architecture better?

The runner-up is an append-only delivery ledger joined by internal request ID, job ID, transport message ID, event type, and event time. It is better when several people investigate cases, when external event names must be normalized, when migrations are likely, or when missing transitions recur often enough to consume feature time. Signed event ingestion, duplicate detection, retention limits, and restricted access then become part of the design. Those controls cost code and operational attention, so the ledger has to earn its place.

For a low-volume product, the lean path is usually enough: a durable job, a correlation ID, a narrow transport adapter, an event trace, and an audited removal decision. Run controlled reset checks against mailboxes you own after changing the worker, message template, event handler, or sending-domain configuration. The check should exercise the real handoff and event path without weakening the generic public response.

The ledger is also not a substitute for recovery security. Keep tokens single-use and expiring, rate-limit requests, avoid automatic account changes before token validation, and do not automatically log the user in after a successful reset; OWASP recommends normal authentication after the password is changed. Delivery evidence answers where the message went. It cannot make a weak token safe.

The final operating rule is plain: inspect broadly, mutate narrowly. Trace the missing transition, preserve recipient intent, and clear a bounce suppression only after the mailbox and cause have been verified. That is enough process to protect users without building a mail platform for sport.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc7208
