# Gaming Signup Evidence: Batch Email, SMS, Pagination, Polling, Recipient Preferences

Short answer: build the Node.js bulk event notification system around one immutable intent per gaming signup, then batch email and SMS work only after resolving recipient preferences and suppression lists. Record every handoff and status transition. A verification link is useful only while it is valid, yet an audit may happen long after delivery.

Batch the work. Keep the evidence.

For a one-person SaaS, I would keep this in Postgres and ship it behind a narrow worker interface. Fewer moving parts protect shipping time. The revenue-per-hour test is blunt: custom queue infrastructure is hard to justify when the differentiated work is account security and the database can already claim work safely.

## How should a Node.js bulk event notification system batch signup messages?

The tempting design is a recipients table plus a loop that sends email and SMS. It is small. It also erases the decisions that matter: which address was selected, which preference version applied, whether a suppression existed, which template revision produced the link, and when responsibility crossed the provider boundary. Treat those as evidence, not debug metadata. A notification intent should capture the signup event ID, recipient ID, channel, normalized destination fingerprint, template revision, preference snapshot, suppression decision, link expiry, creation time, and a stable idempotency key. Keep the raw address in a separately protected field if delivery requires it; routine logs and status views can use the fingerprint. A status such as `sent` is too vague. Use explicit transitions such as `pending`, `claimed`, `submitted`, `delivered`, `failed`, `suppressed`, and `expired`. `submitted` means the provider accepted the handoff. It does not prove inbox or handset delivery. Postmark's transactional email guidance makes the operational distinction between sending and monitoring delivery activity; Twilio's fraud guidance also shows why SMS traffic needs controls before a send is accepted.

Acceptance is not delivery.

The audit trail should be append-only. The current state can be a convenient projection, but each transition needs its own timestamp, source, attempt number, and external message identifier when one exists. This is the concrete constraint that changes the architecture.

## The smallest worker I would ship

Use keyset pagination, not offsets. Offsets can skip or repeat rows as workers update the same result set. A composite cursor over `(created_at, id)` remains stable, while `FOR UPDATE SKIP LOCKED` lets several workers claim different rows without waiting on one another.

I choose 100 rows as a conservative starting batch in the example, not as a universal optimum. Measure claim time, provider latency, and retry pressure before changing it.

The selection step must join the evidence already frozen on the intent. Do not re-read a mutable preference row after claiming work and pretend it was the decision used at creation time. A later opt-out still needs enforcement, so perform a final suppression check immediately before handoff and record its result as a new event.

```ts
import { Pool, PoolClient } from 'pg';

type Intent = {
  id: string;
  channel: 'email' | 'sms';
  destination: string;
  verificationUrl: string;
  expiresAt: Date;
};

type Transport = {
  submit(intent: Intent): Promise<{ externalId: string }>;
  status(externalId: string): Promise<'pending' | 'delivered' | 'failed'>;
};

const pool = new Pool();
const pageSize = 100;

async function claimPage(db: PoolClient): Promise<Intent[]> {
  const result = await db.query<Intent>(`
    WITH candidates AS (
      SELECT id
      FROM notification_intents
      WHERE state = 'pending'
        AND expires_at > now()
      ORDER BY created_at, id
      FOR UPDATE SKIP LOCKED
      LIMIT $1
    )
    UPDATE notification_intents AS n
    SET state = 'claimed', claimed_at = now()
    FROM candidates
    WHERE n.id = candidates.id
    RETURNING n.id, n.channel, n.destination,
              n.verification_url AS "verificationUrl",
              n.expires_at AS "expiresAt"
  `, [pageSize]);
  return result.rows;
}

async function isSuppressed(db: PoolClient, intent: Intent): Promise<boolean> {
  const result = await db.query<{ blocked: boolean }>(`
    SELECT EXISTS (
      SELECT 1 FROM notification_suppressions
      WHERE channel = $1 AND destination = $2
    ) AS blocked
  `, [intent.channel, intent.destination]);
  return result.rows[0].blocked;
}

async function dispatch(transport: Transport, intent: Intent): Promise<void> {
  const db = await pool.connect();
  try {
    if (intent.expiresAt <= new Date() || await isSuppressed(db, intent)) {
      await db.query(
        `UPDATE notification_intents
         SET state = 'suppressed', finished_at = now()
         WHERE id = $1 AND state = 'claimed'`,
        [intent.id],
      );
      return;
    }

    const receipt = await transport.submit(intent);
    await db.query(
      `UPDATE notification_intents
       SET state = 'submitted', external_id = $2, submitted_at = now()
       WHERE id = $1 AND state = 'claimed'`,
      [intent.id, receipt.externalId],
    );
  } finally {
    db.release();
  }
}
```

Keep the claim transaction short: claim and commit, then call the network. Holding a database lock across an external request turns provider latency into database contention. If the process dies after the provider accepts the message but before the update commits, the idempotency key must let the transport reject or reconcile a duplicate. Where a transport offers no idempotent submission contract, mark that limitation explicitly and reconcile before retrying.

The snippet shows the core boundary, not the full schema. In production, insert a transition row in the same transaction as every state change. I would also encrypt destinations at rest and avoid placing verification URLs in general application logs.

## Polling without manufacturing certainty

Status polling is a separate workload. Select only `submitted` rows whose `next_poll_at` has arrived, page by that timestamp and ID, then ask the matching transport for its latest status. Store the raw provider status as evidence and map it to the small internal state machine. Never infer `delivered` merely because no error arrived.

Backoff matters because pending deliveries can outlive a fast worker loop. Schedule the next poll later after each unchanged response, stop when the result is terminal, and stop when the verification link expires. The exact intervals belong in configuration because provider behavior differs; inventing universal numbers would create false precision.

No receipt, no certainty.

Failures split into two classes. A temporary network or rate response can be retried with a recorded attempt. A permanent destination failure should update suppression so the next event is rejected before dispatch. For SMS, add velocity limits around signup attempts and destination patterns. Fraudulent traffic can create real charges and abuse even when the application itself is behaving as coded.

This is where a concise operational dashboard earns its keep. Count intents by channel and state, age of the oldest claim, expired unsent intents, suppression decisions, retry attempts, and records awaiting reconciliation. Alert on stuck age and unexpected state ratios, not on a single generic failure counter.

## Preferences are policy, not presentation

A verification message is transactional, but that label does not erase local law, channel consent, or a recipient's explicit restrictions. Model policy decisions as data with a version and effective time. The worker should receive a resolved channel decision; it should not contain scattered business rules that drift between email and SMS branches.

Use one suppression path for manual blocks, permanent delivery failures, abuse controls, and legally required exclusions, while retaining a reason code and source. Do not delete the evidence when a recipient changes a preference. Record the new decision and its effective time, then make future intents use it. Data retention and access should follow the applicable jurisdiction and the organization's documented policy; there is no honest universal retention period.

A useful pre-deployment test matrix is small:

- email allowed, SMS denied;
- both channels allowed, with email selected by policy;
- destination suppressed after intent creation but before handoff;
- link expired while queued;
- worker exits after provider acceptance;
- two workers claim at the same time;
- provider remains pending across several polls;
- permanent failure creates a suppression decision.

Each test should assert both the outward call and the stored transition history. That second assertion catches the expensive class of bug: the message went out, but the business cannot later explain why.

## What I would change at scale

First, I would partition dispatch from polling so slow status checks cannot starve fresh verification links. Then I would add an outbox boundary if signup creation and notification intent creation span services. The invariant is simple: committing an eligible signup must commit its notification intent exactly once.

At higher volume, partition tables by creation time only after query plans and maintenance show a need. Add channel-specific concurrency controls because email and SMS transports expose different limits and failure semantics. Archive transition evidence under a documented retention policy, but keep the live state table compact enough for predictable claims.

I would outsource transport delivery and keep policy, evidence, and reconciliation inside the application boundary. Transport is undifferentiated until a regulatory or deliverability requirement proves otherwise. Weekly shipping favors the smallest architecture that preserves the audit trail.

The trade-off is more writes: every meaningful decision becomes a record. That is deliberate. For gaming account verification, a durable explanation of consent, suppression, submission, and delivery is worth more than a clever worker that only reports that a loop completed.

## References

- https://postmarkapp.com/guides/transactional-email-best-practices
- https://www.twilio.com/docs/verify/preventing-toll-fraud
