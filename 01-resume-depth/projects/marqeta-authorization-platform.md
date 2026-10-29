# Marqeta — Real-time Payment Authorization Platform (Oct 2021 – Sep 2022)
Role: Staff SWE, Platform Infrastructure · Team of 5 · Remote

## Impact
- Authorization decisions at sub-millisecond latency across millions of daily card events.
- Fraud detection latency: minutes → seconds, via streaming transaction pipelines.
- Java SDK became the standard integration client for internal teams and external partners.

## The story in 2 minutes
Real-time authorization service: routing by merchant category with live balance checks.
Kafka consumer pipelines with exactly-once semantics — partitioned consumer groups sized
to peak throughput, dead-letter handling — feeding live transactions into fraud scoring.
Enrichment rewritten as micro-batched stream against Redis cache (hit rate up, per-record
latency down). Platform services migrated to Kubernetes: autoscaling on consumer lag and
request latency, pod disruption budgets, graceful shutdown. Go tokenization libraries
with format-preserving encryption.

## Hardest decisions / trade-offs
- Exactly-once cost vs at-least-once + idempotent consumers.
- Micro-batching enrichment: latency vs throughput.
- Autoscaling on lag (reactive) vs provisioned headroom (cost).

## Nasty questions
1. "Sub-millisecond" — measured where, and what dominates the budget?
2. Exactly-once in Kafka — what did you actually rely on (transactions, EOS, dedupe)?
3. Fraud scoring moves from minutes to seconds — what was the old path, what changed?
4. A Kafka partition falls behind during peak — system behavior and user impact?
5. Balance check is stale — who eats the loss, and how does the design prevent it?

## Gaps to fill
- [ ] Peak TPS, authorization latency budget end-to-end, uptime target
