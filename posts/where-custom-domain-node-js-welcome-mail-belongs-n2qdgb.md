# Where Custom-Domain Node.js Welcome Mail Belongs: API, Templates, and DKIM

**Short answer:** put transactional welcome mail behind a small queue-backed boundary, render the template in your codebase, and send through a custom-domain email API adapter; a direct API call is the simpler runner-up, but it couples signup latency to mail delivery.

| Shape | Moving parts | Failure boundary | Best fit |
|---|---:|---|---|
| Direct API call in the signup request | Fewest | Signup and mail share one request | A disposable prototype or low-volume internal tool |
| Queue, worker, and API adapter | Moderate | Signup commits before mail is attempted | A small SaaS where welcome mail matters |
| Operate the mail transfer stack | Most | Fully owned | A team with an unusual control or policy requirement |

For a one-person SaaS, the middle row is the useful default. It outsources undifferentiated delivery work without burying business rules inside a vendor integration. The catch is real: a queue adds deployment and observability work. Don't pay that tax before the message matters.

## How should Node.js send a custom-domain welcome email through an API?

Start by separating three decisions that are often mashed into one setup screen. Your application owns the welcome event and the data allowed into the message. A renderer owns the subject, text, and HTML. A transport adapter owns the API request. DNS authentication belongs to the sending domain, not to the template or to a particular route in the application.

That separation is small, but it changes the replacement cost. The signup handler should know that a `welcome.requested` event exists; it shouldn't know an API payload, a remote template identifier, or a retry schedule. Likewise, the template should accept a narrow model such as a display name and a verified app URL. Passing an entire user record is easy. It's also an unnecessary privacy and coupling bet.

Keep the send asynchronous once a missed welcome message would create support work. Commit the account first, enqueue a stable event identifier, and let a worker render and deliver it. Retries then belong to the job system. The transport can change without rewriting signup, and template review remains a normal code review.

This is the revenue-per-hour test: will another hour on mail infrastructure improve onboarding more than another hour on the product? Usually, no. I would own the event, copy, and tests, then outsource transport.

Ship weekly.

## Criterion one: make correctness observable without trusting opens

Define success at each boundary. The signup transaction succeeds when the account and outbox record commit. The worker succeeds when the configured transport accepts the message. Neither event proves that a human read it, so store them as different states and attach the same internal message ID to logs. This gives support a useful trail without pretending a single metric tells the whole story. Open tracking is especially weak as an onboarding success signal. Apple's Mail Privacy Protection can prevent senders from learning whether a recipient opened a message and masks the recipient's IP address. That makes an open-rate alarm a poor proxy for delivery or product activation. Measure the product action instead: did the account reach the next meaningful state after the welcome event? The exact state varies by product, and I'm not sure a universal activation event exists. A deployed app can answer that with its own funnel data. Test the parts you own. Snapshot both text and HTML output with fixed input. Assert that user-controlled values are escaped in HTML. Exercise the worker with a fake transport that records calls, then verify that the same job ID cannot create two application-level send attempts after a retry. In staging, use addresses and domains reserved for testing by your own environment; don't turn a real customer's inbox into a deployment check. Short logs beat a dashboard full of opens. Record the event ID, template version, recipient-domain hash (rather than a raw address), attempt count, transport status class, and final state. Avoid message bodies in routine logs. If the worker exhausts its policy, move the job to a reviewable terminal state rather than retrying forever. That operational path matters more than a clever template engine because it tells you which account needs attention and why.

The queue — not signup — owns recovery.

## Criterion two: treat DKIM and DMARC as deployment inputs

A custom `From` domain is an identity boundary. DMARC evaluates whether authenticated identifiers align with the visible author domain, using SPF and DKIM results as described in RFC 7489. So the deployment checklist must cover more than adding a DKIM record and seeing a green badge: choose the author domain, publish the records supplied for that domain, verify alignment, and keep those DNS choices under the same change discipline as production configuration.

The practical test is simple. Send from the exact domain and stream that production will use, then inspect the received message's authentication results. Test the text alternative, links, reply address, and display name at the same time. A passing test for one subdomain doesn't establish alignment for every other author domain. Domain boundaries are the unit of the check.

DMARC aggregate reports can provide visibility into authentication results for mail claiming to come from the domain. They are operational input, not an onboarding KPI. Decide who receives them, how often they are reviewed, and what change would trigger investigation before tightening policy. RFC 7489 also discusses gradual policy deployment; use evidence from your own legitimate streams rather than jumping straight to a strict posture.

Keep marketing and transactional streams distinct in your application model even if they initially share transport plumbing. Their consent rules, message cadence, templates, and failure impact differ. Welcome mail should be triggered by an explicit product event, with the destination captured for that event. This makes later policy or transport changes far less invasive.

DNS changes can outlive the code that requested them. Write down the selector owner, sending purpose, and removal condition next to the deployment record. When a transport is retired, remove its application credentials first and retire obsolete DNS authorization through a reviewed change. Boring work. Necessary work.

## A focused TypeScript send example

The example below keeps the external contract generic on purpose. `MAIL_API_URL` is the full endpoint documented by the chosen transport; the application doesn't invent a route. The adapter sends a conventional JSON object, but its final field mapping should be changed in one place to match that documented contract.

```ts
type WelcomeJob = {
  eventId: string;
  to: string;
  displayName: string;
  appUrl: string;
};

type RenderedMail = {
  subject: string;
  text: string;
  html: string;
};

interface MailTransport {
  send(job: WelcomeJob, mail: RenderedMail): Promise<void>;
}

const escapeHtml = (value: string): string =>
  value.replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#39;");

function renderWelcome(job: WelcomeJob): RenderedMail {
  const name = escapeHtml(job.displayName);
  const url = escapeHtml(job.appUrl);

  return {
    subject: "Welcome — your workspace is ready",
    text: `Hi ${job.displayName},\n\nOpen your workspace: ${job.appUrl}`,
    html: `<p>Hi ${name},</p><p><a href="${url}">Open your workspace</a></p>`,
  };
}

class JsonApiTransport implements MailTransport {
  constructor(
    private readonly endpoint: string,
    private readonly token: string,
    private readonly from: string,
  ) {}

  async send(job: WelcomeJob, mail: RenderedMail): Promise<void> {
    const response = await fetch(this.endpoint, {
      method: "POST",
      headers: {
        authorization: `Bearer ${this.token}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        from: this.from,
        to: job.to,
        subject: mail.subject,
        text: mail.text,
        html: mail.html,
        eventId: job.eventId,
      }),
    });

    if (!response.ok) {
      throw new Error(`Mail transport rejected the request: ${response.status}`);
    }
  }
}

async function handleWelcome(
  job: WelcomeJob,
  transport: MailTransport,
): Promise<void> {
  await transport.send(job, renderWelcome(job));
}
```

The worker should load secrets from its deployment environment, instantiate the adapter, and call `handleWelcome`. The queue policy decides when to retry; the adapter only reports success or failure. Before adopting a transport, verify its documented request schema, authentication header, timeout behavior, retry guidance, message-size limits, and duplicate-send protections. If its contract uses `429` for rate limiting, honor the documented delay with queue-level backoff; if it supports an idempotency key, derive that key from `eventId` and send it under the exact header the contract names. Your mileage may vary because these are transport contract choices, not properties of Node.js.

There is one sharp edge in this compact sample: HTML escaping isn't URL validation. Build `appUrl` from a trusted application origin plus an opaque server-created path, rather than accepting an arbitrary URL from a user profile. Keep the plain-text body too. Templates should be versioned with code, and a template change should run the same tests as a worker change.

## When is the direct API call the better runner-up?

Stick with a direct call when the project is an early prototype, mail is noncritical, and the team is prepared to show signup success even if the send attempt needs separate follow-up. It has fewer components, which can be the right revenue-per-hour choice while demand is still uncertain. Put the call behind the same `MailTransport` interface so the queue can arrive without changing template or signup semantics.

The direct shape is not suitable when a slow transport can consume the signup request budget, when retries could duplicate side effects, or when support needs an independent delivery trail. Move to an outbox and worker at that point. Conversely, operating a mail transfer stack can be justified by a control, regulatory, or integration constraint that an external API cannot meet, but it demands specialized operational ownership. That is rarely compatible with a solo builder shipping every week.

Choose the smallest boundary that preserves the next likely change. For most small products with meaningful transactional mail, that means application-owned events and templates, an asynchronous worker, standards-aware domain configuration, and a replaceable API adapter. No leaderboard required.

## References

- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
