# Project 1 — Serving Control Plane: Custom Operator → KServe + GPU Scheduling

## Goal

Rebuild the core of the Current Employer model serving platform: a Kubernetes
operator that lets ML teams deploy models without writing manifests, a migration
to KServe with both paths running side-by-side, and admission-time GPU
scheduling that raises utilization from ~40% to ~80%.

## Resume coverage

- Lead design and delivery of the model serving platform used by internal ML
  teams; own architecture, delivery, and on-call
- Python Kubernetes operator reconciling an in-house InferenceEndpoint custom
  resource into Deployments, Services, autoscalers, RBAC, and ingress
- Migration to KServe with both paths running side-by-side in production with
  no serving interruption
- Admission-time GPU scheduling: MIG slice, time-sliced GPU, or dedicated full
  GPU chosen by model size and SLA tier — GPU utilization ~40% → ~80%

## What to build

1. **InferenceEndpoint operator (kopf)**
   - CRD: model reference, SLA tier, resource profile, autoscaling bounds
   - Reconciliation into Deployment, Service, HPA, RBAC, Ingress
   - Status conditions, finalizers, idempotent reconcile, leader election
2. **KServe path**
   - Same endpoints expressed as InferenceServices
   - Traffic split so legacy operator path and KServe path run side-by-side
   - Cutover and rollback procedure per endpoint, no downtime
3. **Admission GPU scheduler (mutating/validating webhook)**
   - Placement decision at admission: MIG slice / time-sliced / dedicated
   - Inputs: model size, SLA tier, current fleet fragmentation
   - Decision recorded as an annotation + metric, so placement is auditable

## Production-like scale targets

- 20–30 model endpoints across 3 SLA tiers
- Workload replay comparing naive dedicated-GPU allocation vs the scheduler,
  reporting utilization the same way for both (target: reproduce the
  ~40% → ~80% shape, not just claim it)
- One MIG-capable cloud GPU node (for example an A100 instance) for real MIG
  profiles; remaining fleet as simulated GPU nodes — labelled honestly

## Design → production steps

- [ ] Design doc: CRD schema, reconcile loop, webhook decision flow, failure modes
- [ ] Operator with unit + envtest coverage for reconcile idempotency
- [ ] Helm chart + GitOps deployment, upgrade and rollback tested
- [ ] KServe migration runbook, executed end-to-end on a subset of endpoints
- [ ] Load/replay harness producing the utilization comparison
- [ ] Dashboards and alerts for reconcile errors, admission latency, placement mix
- [ ] On-call runbook: stuck endpoint, webhook outage (fail-open vs fail-closed), bad cutover

## Knowledge gaps this closes

- MIG profile geometry, GPU device plugin behaviour, and what fragmentation
  does to placement over time
- Admission webhook latency budgets and failure policy trade-offs
- KServe internals: storage initializer, KPA autoscaling, predictor/transformer split
- Operator edge cases: finalizer deadlocks, status update conflicts, watch storms

## Completion proof

- Repo with operator, webhook, Helm charts, and replay harness
- Published utilization comparison with the raw numbers behind it
- Migration demo: endpoint serving traffic before, during, and after cutover
