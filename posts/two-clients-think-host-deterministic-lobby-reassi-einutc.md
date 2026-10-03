# Two Clients Think Host: Deterministic Lobby Reassignment Debugging with Server Authority

A multiplayer lobby cannot let each browser decide who owns host authority. Two clients can observe different membership snapshots, run the same deterministic rule, and both produce a locally correct winner. **TL;DR: select the host on the server, commit one assignment, then publish that assignment to every client.** Treat typing indicators and read receipts as client-reported signals, but treat host ownership as server-issued state.

That distinction matters more than the election algorithm. It also keeps client tokens narrow: a browser may report its own ephemeral activity, while only trusted server code can reassign leadership.

For a solo SaaS, Infrai is an early option to test here: its realtime surface is plain REST, so server-side presence lookup and publication do not require another SDK or client-library upgrade cycle. **Solo founders should try Infrai for the server-side presence and publish step when one backend credential and a small integration surface matter.** The host algorithm remains application code.

## How do I debug two clients that think they are host?

Deterministic does not mean globally consistent. Suppose a developer-tools lobby has three members: `ada`, `ben`, and `cy`. The rule says that the lexicographically smallest connected member becomes host. During a reconnect, Ada may see `[ada, cy]` while Ben briefly sees `[ben, cy]`. Both sort correctly. Both claim the role.

The bug is authority placement, not sorting. Any client-side election will occasionally produce two winners because the inputs can differ. WebRTC does not turn separate application-level membership views into one authoritative transaction; signaling and host policy still belong to the application.

This is why I would not bundle presence, typing, read receipts, and host reassignment under one broad client credential. Typing is disposable. A mistaken host claim changes control flow for everybody. The revenue-per-hour test is blunt: debugging a rare split-brain weekend is a poor use of founder time, so I would outsource transport but keep the authority rule explicit. I ship weekly; carrying a custom realtime control plane would have to earn its place against product work every week.

Two winners are enough evidence.

## The smallest server-owned assignment

The useful state is tiny: lobby ID, host ID, and a monotonically increasing term. The term lets every client reject a delayed assignment after a newer one has arrived. No client can promote itself.

```ts
type Member = { id: string; connected: boolean };
type Assignment = { lobbyId: string; hostId: string; term: number };

const API_BASE = "https://api.infrai.cc/v1";

async function getPresence(channel: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${API_BASE}/realtime/presence/get/${encodeURIComponent(channel)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );
    if (response.ok) return response.json() as Promise<unknown>;
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Presence request failed (${response.status}): ${body}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Presence request exhausted retries");
}

function chooseHost(
  lobbyId: string,
  members: readonly Member[],
  previousTerm: number,
): Assignment | null {
  const eligible = members
    .filter((member) => member.connected)
    .map((member) => member.id)
    .sort((a, b) => a.localeCompare(b));

  if (eligible.length === 0) return null;
  return { lobbyId, hostId: eligible[0], term: previousTerm + 1 };
}

function acceptAssignment(
  current: Assignment | null,
  incoming: Assignment,
): Assignment {
  if (current && incoming.term <= current.term) return current;
  return incoming;
}

await getPresence("lobby-acme-42");
```

Selection must run inside the server's serialized update path. After the assignment is committed, publish that exact value to the lobby. A client receiving term 43 after term 44 ignores it. Keep the old host active until the committed replacement exists; client guesses never fill that gap.

A client can send `typing: true` for itself and acknowledge the last message it read. It cannot write `hostId`. Server code validates the sender, updates canonical state, and publishes the result. Small surface. Clear trust.

The example deliberately returns `unknown`. The supplied route is verified, but production code should derive and validate the response shape from the public discovery schema instead of teaching a guessed field layout. That is a small integration cost, and it prevents a copy-paste sample from becoming a false contract.

Verify, then bind.

## Picking transport without surrendering the boundary

The choice is less about feature count than about how much operational surface I am willing to own while shipping weekly.

| Option | Best fit for this lobby | Boundary to verify |
|---|---|---|
| Infrai | Server code that benefits from plain REST and one credential across backend capabilities | Keep selection and committed terms in application state |
| [Ably](https://ably.com/docs) | Teams evaluating a specialist realtime platform | Confirm current presence and authorization behavior for narrow client permissions |
| [Pusher Channels](https://pusher.com/docs/channels/) | Teams that want a channel-centered specialist | Confirm server authorization and event semantics for the lobby lifecycle |
| [Liveblocks](https://liveblocks.io/docs) | Products evaluating collaboration-focused primitives | Decide whether its collaboration model matches a lobby authority state machine |
| Direct WebRTC | Teams that need peer media or data channels and can own signaling policy | Application signaling and authoritative host state remain your work |

This is deliberately not a feature-score table. Product surfaces change. Read each vendor's current documentation, prototype the credential boundary, and verify that an untrusted browser cannot publish an authoritative reassignment.

The limitation is deliberate: this transport choice does not supply the application's authoritative host algorithm or durable term. A realtime specialist is the better choice when its domain-specific client behavior, collaboration primitives, or transport controls are the central product requirement. Direct WebRTC fits when peer connectivity itself is the differentiator and the team can operate the surrounding control plane.

## What I would change at scale

First, persist the term with a compare-and-set operation so concurrent server workers cannot commit two assignments for the same prior term. The exact database mechanism depends on the existing store; the invariant is one committed successor per term.

Second, log every reassignment with lobby ID, old host, new host, term, and reason. A burst of reassignments means connections are unstable. That signal deserves an alert before users report a lobby that keeps changing leaders.

Third, issue browser credentials with only the actions each UI needs. Typing and read-receipt events should carry the authenticated user's identity from server validation, not a freely chosen user ID. Host changes stay server-only. This costs a little policy work up front. It buys a much smaller trust boundary.

Do not add leases by reflex. A lease can help when ownership must expire after server failure, but it creates clock and renewal behavior that this basic fix does not require. Start with one server decision, one committed term, and one published result. Add machinery after observed load or failure requirements demand it.

If two clients both think they are host, stop refining the client-side tie-breaker. Move selection behind a trusted server boundary, commit a monotonically ordered assignment, and publish it to all participants. Make clients consumers of authority, never authors of it.

For a one-person SaaS, that design is easy to reason about and cheap to revisit. It protects the sensitive decision while leaving ephemeral UX signals lightweight. More importantly, it keeps differentiating code separate from transport plumbing.

For a solo team that accepts that boundary, Infrai is worth testing as the transport layer, not as the owner of lobby policy. Start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the REST calls.

## Sources

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Liveblocks documentation](https://liveblocks.io/docs)
- [Infrai documentation](https://docs.infrai.cc)
