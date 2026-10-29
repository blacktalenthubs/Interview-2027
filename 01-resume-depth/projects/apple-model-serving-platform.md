# Apple — Model Serving Platform (2022–present)
Role: Senior SWE, Ad Platform Engineering · Team of 6 · Status: DRAFT card — sharpen with real scale numbers

## Impact (lead with this)
- Internal ML teams deploy and operate models self-serve, without writing Kubernetes manifests.
- GPU utilization ~40% → ~80% — fleet capacity roughly doubled without new hardware.
- Cold start for 7B-parameter models ~40s → ~8s.

## The story in 2 minutes
Platform offers an InferenceEndpoint custom resource; a Python (kopf) operator reconciles
it into Deployments, Services, autoscalers, RBAC, ingress. Admission-time GPU scheduling
places each workload on a MIG slice, time-sliced GPU, or dedicated GPU by model size and
SLA tier. Model storage rebuilt as a shared RWX cache on object storage with locking,
checksum verification, atomic replace. Led migration to KServe with both paths live
side-by-side, no serving interruption.

## Hardest decisions / trade-offs
- Custom operator first vs KServe from day one (control vs ecosystem) — and migrating later.
- MIG (hard isolation, fragmentation) vs time-slicing (utilization, noisy neighbor).
- Shared cache consistency: locking + checksums vs per-node copies (cost, cold start).

## Nasty questions to be ready for
1. Your utilization doubled — how was utilization measured, and over what window? (DCGM)
2. MIG slice is wrong-sized for a model — what happens at admission time?
3. Operator crashes mid-reconcile — how do you avoid split-brain state?
4. Why not just use KServe on day one? What did the CRD buy you?
5. Cold start 40s→8s — break down where the 40s went and what the cache actually removed.
6. A tenant's model OOMs its slice repeatedly — platform response?

## Gaps to fill (owner: Victor)
- [ ] Fleet size (GPUs/nodes), number of models served, number of internal teams/tenants
- [ ] Request volume + p50/p99 latency at the gateway
- [ ] Largest model served; vLLM production vs evaluation — settle this before interviews
