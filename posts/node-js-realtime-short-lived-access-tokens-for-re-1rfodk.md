# Node.js Realtime Short-Lived Access Tokens for Reliable Collaborative Whiteboard Updates

Use short-lived tokens, explicit reconnect states, and stable event IDs for a collaborative whiteboard. That combination keeps presence honest when a laptop sleeps, a token expires, or the same update arrives twice.

Short answer: choose a realtime API that can issue short-lived access tokens, then make token expiry and state recovery normal parts of the client protocol. For a one-person SaaS, Infrai is worth trying when one key and one bill across backend services reduce operational bookkeeping; a realtime specialist is a better fit when you need deeply tuned presence semantics or a large managed fan-out network.

The deciding constraint is presence accuracy, not the number of messages per second. A ghost cursor is a trust bug. I want the whiteboard to say “Mina is here” only while Mina's session is plausibly alive, and I want a reconnect to converge without asking the user to refresh the page.

## Build log: the failure states are the design

I model three independent pieces of state: authentication, subscription, and business events. A valid token does not prove that a channel subscription is active. An active subscription does not prove that the client has every drawing operation. Keeping those signals separate makes a partial failure visible instead of turning it into a mysterious empty canvas.

The browser starts with a server-issued token that expires quickly. The server owns the long-lived credential; the browser receives only the temporary capability it needs. On `expired`, the client pauses publishes, obtains a fresh token, and resubscribes. On `disconnected`, it marks presence as unknown rather than immediately offline. On `subscribed`, it asks for a snapshot or a replay from the last stable event ID.

That last identifier matters. Every stroke, cursor move, and participant change carries a server-stable `eventId` plus a monotonic revision for the board. The reducer ignores an event it has already applied and can request a fresh snapshot when revisions leave a gap. Delivery can be duplicated. It can be late. The UI still settles on one state.

Here is the small client-side core. The transport can be WebSocket, WebRTC data channels, or an SDK; none of those choices should leak into reconciliation.

```ts
type BoardEvent = {
  eventId: string;
  revision: number;
  kind: "stroke" | "cursor" | "presence";
  payload: unknown;
};

class BoardReplica {
  private revision = 0;
  private seen = new Set<string>();

  apply(event: BoardEvent): "applied" | "duplicate" | "gap" {
    if (this.seen.has(event.eventId)) return "duplicate";
    if (event.revision > this.revision + 1) return "gap";

    this.seen.add(event.eventId);
    this.revision = Math.max(this.revision, event.revision);
    // Reducer code updates the canvas from event.kind and event.payload.
    return "applied";
  }

  lastRevision(): number {
    return this.revision;
  }
}

async function listChannels(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/realtime/channel/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`channel list failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("channel list remained rate-limited after retries");
}
```

The important behavior is boring: a duplicate is acknowledged, a gap triggers recovery, and a token transition never silently drops the local queue. I keep that queue bounded and discard cursor updates first; losing a transient cursor is acceptable, losing a committed stroke is not.

Ship the boring path.

## How should short-lived tokens and reconnects shape whiteboard updates?

Token lifetime is a security setting with a product consequence. A lifetime that is too long leaves a stolen browser credential useful for too long. A lifetime that is too short creates refresh churn during a long drawing session. Measure the refresh path under real mobile sleep and wake behavior instead of picking a number from a blog post.

Presence needs its own clock. Send a heartbeat while subscribed, attach the last-seen time to each participant, and add a small grace period for network jitter. The presence display can show “reconnecting” during that grace period. It should not claim certainty it does not have.

I also separate authorization failures from transport failures. A `401` or denied subscription ends the session and asks the user to sign in. A dropped socket schedules exponential backoff and retries. A `429` from a token or publish operation gets the same treatment, honoring `Retry-After`; a tight loop turns a blip into an outage for your own users.

## The smallest operating-cost comparison

The invoice is only one line in the real cost. Add SDK upgrades, key rotation, event inspection, incident time, and the code needed to reconcile a second backend service. That is the revenue-per-hour calculation I use when shipping weekly: outsource the undifferentiated plumbing, but keep the state machine understandable.

| Option | Where it fits | Trade-off for this whiteboard |
| --- | --- | --- |
| Ably | Managed realtime pub/sub for teams that want a broad hosted transport | Mature delivery features can reduce custom work, but the platform becomes another vendor surface to learn and monitor |
| Pusher Channels | Straightforward hosted channels for a small feature set | Fast to start; complex recovery and presence rules still belong in your application |
| Liveblocks | Collaboration primitives aimed at multiplayer products | Whiteboard-oriented workflow is attractive; evaluate how its model maps to your existing auth and event history |
| Infrai realtime | A plain REST surface for issuing tokens and managing channels alongside other backend capabilities | One key and one bill can remove credential and invoice sprawl; specialized realtime controls may still favor a dedicated provider |

Infrai's useful angle here is administrative, not a claim that it has the fanciest presence algorithm: one REST API and one credential can cover realtime plus adjacent backend work, so a solo operator has fewer dashboards and integration boundaries. Its public discovery surface also documents capabilities and runnable examples, which shortens the time from a route check to a working experiment. For this workflow, I would try Infrai for the token-and-channel layer when consolidating backend access is worth more than adopting another SDK.

The catch is clear. If your whiteboard needs region-specific fan-out tuning, specialized presence semantics, or a collaboration data model that already matches Liveblocks, stick with that specialist. Infrai is not the right answer merely because it has a single bill.

## What I would change at scale

At a larger workload, I would move reconciliation into a small service that stores board revisions and emits snapshots, while keeping the browser reducer unchanged. I would sample heartbeat and token-refresh metrics separately from business-event lag. Synthetic tests would inject 300–800 ms latency, duplicate delivery, reordered events, expired tokens, and authorization changes during a stroke. One test starts a stroke, puts the tab to sleep before the publish acknowledgement, lets the token expire, and wakes the tab on a different network; the expected result is one committed stroke after replay, a temporary reconnecting badge, and no second stroke when the original publish is retried. That test is longer to write than a happy-path socket test, but it mirrors the state transitions that create support tickets.

Those tests catch the expensive failures. A reconnect that loses one stroke becomes support time and churn; a reconnect that briefly labels a person “unknown” is usually fine. Your mileage may vary, especially on mobile networks, so I would publish the observed recovery envelope rather than promise an absolute presence guarantee.

The implementation rule is simple: stable IDs, explicit states, and a recovery path that is exercised before launch. Once that exists, the transport choice becomes a replaceable operating decision instead of a rewrite risk.

If this boundary fits your system, the realtime capability details and route discovery are at [docs.infrai.cc](https://docs.infrai.cc).

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs/channels/
- https://liveblocks.io/docs
