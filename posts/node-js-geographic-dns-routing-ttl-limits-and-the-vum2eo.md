# Node.js Geographic DNS Routing: TTL Limits and the Right Layer (and Why)

**Short answer:** A regional tenant hostname should be a routing hint, not your failover control plane. For a fintech service, I would keep tenant onboarding and zone ownership in the application, publish a short-lived DNS answer only where coarse geography is acceptable, and put health-aware failover at the request-routing layer. DNS caching means a resolver can keep serving an old answer after your control plane changes it. That is the limit to design around.

For tenant subdomains, platform-owned zones are the least complex option when the platform creates names such as `acme.us.example.test`. Customer-owned zones are the right runner-up when the customer must keep the parent domain, certificate policy, and DNS authority. The decision is mostly about who can change records during an incident.

| Choice | Good fit | Main operational cost |
| --- | --- | --- |
| Platform-owned zone | Automatic tenant provisioning and a uniform hostname policy | The platform owns delegation, certificates, and DNS changes |
| Customer-owned zone | A bank needs its own domain and DNS governance | Delegation, verification, certificate issuance, and support cross the customer boundary |
| DNS-based regional hint | Coarse locality or low-frequency steering | Cached answers can outlive the intended routing decision |
| Request-layer routing | Health-aware failover and explicit policy | The routing layer needs probes, state, and a second operational system |

I would choose the platform-owned zone for the default tenant path, then expose customer-owned zones as a boundary with explicit verification. I would not call a DNS answer a health check.

No magic TTL.

## What does DNS actually know about a region?

DNS can return different answers based on information available to the authoritative service or its routing policy. That does not mean the authoritative service sees every end user's location. Recursive resolvers cache answers and serve them to later clients, so the answer may reflect the resolver's location, an EDNS Client Subnet policy, or no useful geographic signal at all. The application must treat the result as approximate.

The second approximation is time. A TTL is a cache lifetime hint for an RRset. It is not a remote delete. Lowering the TTL before a planned change can reduce the future cache window, but it cannot erase an answer already cached under an earlier TTL. During a regional outage, a newly published record therefore has a propagation tail that is bounded by caches and resolver behavior, not by the speed of your deploy job.

That makes DNS a poor place to encode a promise like “all traffic leaves region A within 30 seconds.” It can support a broad preference, such as sending most new lookups toward a nearby region. It cannot provide per-request certainty after an answer has been cached.

The tempting assumption is that a 60-second TTL creates a 60-second failover guarantee. It does not.

That is the operational trap.

Keep the recovery rule boring.

## Why is the zone boundary the real tenant decision?

The easy part is generating `tenant-482.eu.example.test`. The hard part is deciding who has authority to change what that name means. With a platform-owned zone, one service can create the record, issue or attach the certificate, and remove the tenant during an offboarding workflow. That is a compact control surface. It also makes the platform responsible for every DNS and certificate failure in the path.

Customer-owned zones reverse those obligations. A customer may delegate `payments.customer.example` to nameservers you operate, place a CNAME at a specific label, or use another documented arrangement. The exact mechanism depends on the customer's DNS provider and policy. Verification must prove control of the intended name before the platform accepts it; a display-name match is not proof.

This is where fintech workflows become less forgiving. A typo in a delegated label can send verification to the wrong place. An abandoned record can keep pointing at a decommissioned tenant. A customer may also require DNS changes to pass an internal review window. The onboarding state machine needs explicit states for requested, verified, active, suspended, and revoked. DNS publication should follow that state, not lead it.

I keep the tenant's canonical identity separate from its hostname. The database stores the tenant ID, ownership mode, verified hostname, region preference, and certificate status. The DNS adapter receives a desired record set and an idempotency key. It does not decide whether a tenant is allowed to become active. That decision stays in the application.

## Is geographic DNS routing the right layer for failover?

For coarse steering, yes. For a deadline, no. The distinction matters because the same tenant can have a warm connection, a cached answer, and an application session that each outlive a DNS change. A two-region deployment can therefore show healthy DNS observations while requests still reach the old region. The fix is to make the request layer enforce health and keep DNS responsible for naming and broad entry-point selection.

## How should a Node.js control path publish tenant DNS?

The useful boundary is a generic DNS publisher. It accepts a desired state, records what was requested, and lets reconciliation retry safely. The sample leaves provider calls behind an interface because the important behavior is ownership and convergence, not a vendor SDK.

```ts
type ZoneOwnership = "platform" | "customer";
type TenantDnsState = "requested" | "verified" | "active" | "suspended" | "revoked";

type TenantRoute = {
  tenantId: string;
  hostname: string;
  ownership: ZoneOwnership;
  state: TenantDnsState;
  region: string;
  target: string;
  ttlSeconds: number;
};

interface DnsPublisher {
  apply(route: TenantRoute, idempotencyKey: string): Promise<void>;
  remove(route: TenantRoute, idempotencyKey: string): Promise<void>;
}

async function reconcileTenant(route: TenantRoute, dns: DnsPublisher) {
  if (route.state === "active") {
    await dns.apply(route, `tenant:${route.tenantId}:active`);
    return;
  }

  if (route.state === "suspended" || route.state === "revoked") {
    await dns.remove(route, `tenant:${route.tenantId}:${route.state}`);
  }
}
```

The code deliberately does not switch a tenant's region on a failed probe. A health system should make that decision after checking the application, dependencies, and recovery policy. The DNS reconciler should publish the resulting desired state and emit an audit record containing the tenant ID, hostname, old target, new target, actor, and reason.

A few tests matter more than a large integration suite: a customer-owned name cannot enter `active` before verification; a revoked tenant produces a removal operation; retries with the same idempotency key converge; and a stale reconciliation event cannot overwrite a newer state. I would also test the cache window as a property of the incident runbook, not as a promise hidden in a unit test.

## When should the runner-up win?

Choose customer-owned zones when the hostname is part of the customer's contractual identity or when their security team must retain DNS authority. Accept the slower onboarding path. Make delegation and verification observable, with an owner and an expiry for every pending request.

Use DNS-based regional hints when a wrong-region request is inconvenient but safe. Examples include a read-heavy dashboard or a stateless landing endpoint. Keep the answer usable in every region, because a cached answer can send a client somewhere you no longer prefer.

Use request-layer routing when writes, sessions, regulated records, or failover deadlines are involved. The routing layer can evaluate health on the request path or at a controlled edge, reject a region that is unhealthy, and apply a policy that is visible in logs. It costs more operational work. That is a fair trade when the revenue-per-hour cost of a silent regional split is high.

The practical pattern is hybrid: DNS selects an entry point, and the entry point selects a healthy region. A DNS change is then a coarse traffic-shaping action, not the only recovery mechanism. The limitation is intentional: DNS cannot revoke every cached answer on demand, so a design that requires that guarantee belongs at a layer with request-level state.

## What should the weekly operating method measure?

First, publish only targets that can serve the tenant safely. A DNS record should not be the thing that makes a tenant active. Second, monitor from outside the authoritative system: resolve through several recursive resolvers, record the observed TTL and answer, and compare it with the desired state. Third, give every regional change a rollback record and a human-readable incident reason.

The useful alert is not “the DNS API returned success.” It is “resolver observations still disagree with the desired tenant route after the expected cache window.” That distinction catches stale data, delegation mistakes, and a control plane that wrote the wrong zone.

For a tenant move, the runbook should name the old and new regions, the record version, the expected cache window, and the point at which request-layer routing takes over. It should also say what happens to writes already accepted by the old region. A DNS edit cannot answer that question. The application needs a replication and session policy, and the incident commander needs an explicit choice: drain, reject, or continue serving a degraded read path. Writing these decisions down takes less time than reconstructing them while a customer is watching a stale hostname resolve.\n"

I would ship the platform-owned path first, because one person can operate it weekly without turning every onboarding into a cross-company project. Customer-owned zones remain a deliberate exception with stronger checks. DNS handles naming and coarse steering. Health-aware routing handles failover. Keeping those jobs separate is the decision.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc1034
- https://www.rfc-editor.org/rfc/rfc1035
- https://developer.mozilla.org/en-US/docs/Glossary/TTL
