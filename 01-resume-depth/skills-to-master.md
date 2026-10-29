# Top Skills to Master — derived from the final resume (Oct 2026)

Ranked by how hard Staff-level loops will probe them.

## 1. Kubernetes platform engineering
Operators & CRDs (kopf), reconciliation loops, admission control, RBAC, autoscaling,
pod disruption budgets. Depth target: explain your InferenceEndpoint operator's
reconcile flow, error handling, and what you would change at 10x scale.

## 2. GPU / inference serving
MIG vs time-slicing vs dedicated GPUs (when each wins), cold-start anatomy
(image pull, weight load, graph warm-up), KServe, vLLM basics (continuous batching,
PagedAttention at concept level), quantization effects (GGUF Q4–Q8), eval gating.
Depth target: capacity math — given model size + QPS + latency SLO, size a fleet.

## 3. Distributed systems core
Delivery semantics (at-least-once / at-most-once / exactly-once and what it really
costs), idempotency keys, partitioning & ordering, retries with backoff, DLQs,
backpressure, consensus basics, CAP as a trade-off not a slogan.

## 4. Streaming + batch pipelines
Kafka internals (partitions, consumer groups, offsets, ISR), Spark Structured
Streaming, Redis Streams, CDC patterns, late & out-of-order events, schema evolution,
batch/streaming reconciliation (your Dimension Count work is a live example).

## 5. Authorization & multi-tenancy
Zanzibar model & SpiceDB (schema, relationship writes, permission checks at gateway),
Envoy ext_authz, OAuth 2.0 flows, short-lived credentials, tenant isolation patterns,
versioned API contracts.

## 6. Reliability & observability
SLO/error budgets, Prometheus/OTel instrumentation, DCGM & eBPF basics, alert design,
capacity planning from metrics, multi-region DR patterns (active-passive, failover
testing), incident narratives.

## 7. Payments domain depth
Authorization flow end-to-end (cardholder → merchant → acquirer → network → issuer),
tokenization (network tokens, format-preserving encryption), issuer decisioning,
clearing/settlement vs authorization, latency budgets, why exactly-once matters
financially. Decisive in fintech loops; differentiator everywhere else.

## Languages for interviews
Python (primary), Go (secondary — Temporal/Confluent-style loops), Java (read fluency;
your Visa/Marqeta code was Java).
