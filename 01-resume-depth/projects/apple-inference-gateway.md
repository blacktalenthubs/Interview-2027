# Apple — Multi-tenant Inference Gateway & Authorization
Status: DRAFT card

## Impact
- Every inference API call is authenticated, tenant-tiered, and fine-grained authorized —
  multi-tenancy without each team building its own plumbing.

## The story in 2 minutes
Envoy gateway with ext_authz resolving tenant identity and SLA tier; SpiceDB
(Zanzibar-style) permission checks on every call — schema and relationship writes owned
by me on resource creation across inference servers and models; short-lived credentials.
Inference service: lifecycle endpoints, Redis Streams event architecture for server state
transitions, KServe runtime health reporting back to the platform.

## Hardest decisions / trade-offs
- AuthZ check on every call vs cached decisions (freshness vs latency).
- Relationship writes at creation time vs lazy evaluation.
- Event-driven state (Redis Streams) vs polling runtimes.

## Nasty questions
1. SpiceDB check adds latency to every call — how much, and how did you bound it?
2. A permission is revoked — how fast does the gateway stop honoring it?
3. Tenant A spikes — how does the gateway protect tenant B's SLA tier?
4. Why Zanzibar-style instead of RBAC/ABAC here?
5. Redis Streams consumer dies mid-event — what guarantees state convergence?

## Gaps to fill
- [ ] Gateway request rate, authZ check latency budget actually achieved
