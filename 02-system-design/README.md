# Layer 2 — System Design (Staff Level)

## The frame (use every time, ~45 min)
1. **Clarify (5')** — functional reqs, non-functional reqs (scale, latency, consistency,
   availability), scope cuts stated out loud.
2. **Estimate (3')** — QPS (avg/peak), storage/day, bandwidth. Numbers drive design.
3. **API & data model (7')** — endpoints, entities, storage choice per entity with reason.
4. **High-level design (10')** — draw the happy path end-to-end.
5. **Deep dive (15')** — the 1–2 hardest components; this is where Staff is decided.
6. **Trade-offs & evolution (5')** — bottlenecks, failure modes, what you'd build in v2,
   how it scales 10x, cost notes.

Staff signals: you drive the conversation, name trade-offs before being asked, design
for operability (metrics, alerts, backfills, migrations), and connect design to org
(teams, ownership boundaries, rollout plan).

## Practice set (mapped to Layer 1 — practice doubles as resume defense)
- [ ] ML inference platform (multi-tenant, GPU scheduling) — current-employer serving
- [ ] Card authorization system (latency budget, exactly-once) — Marqeta/Visa
- [ ] Streaming event pipeline with fraud scoring — Marqeta
- [ ] Model/artifact catalog with lifecycle gates — current-employer catalog
- [ ] Multi-tenant API gateway with fine-grained authZ — current-employer gateway
- [ ] Rate limiter / quota service (classic, know it cold)
- [ ] Metrics & observability pipeline (high-cardinality ingestion)
- [ ] Workflow orchestration / job scheduler (Argo/Temporal-style)
- [ ] URL shortener / pastebin (warm-up classic)
- [ ] News feed / notification system (breadth classic)

Write each design as `design-<name>.md` in this folder with the frame filled in,
plus a "questions I got stuck on" section.
