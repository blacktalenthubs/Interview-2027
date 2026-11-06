# Project 4 — Runtime Performance + Observability: Model Cache, Cold Start, GPU Telemetry

## Goal

Rebuild the runtime layer under the serving platform: the shared model cache
that cut 7B-model cold start from ~40 seconds to ~8 seconds, the multi-tenant
inference gateway, and the GPU observability stack whose metrics drive
capacity planning, alerting, and incident response.

## Resume coverage

- Model storage rebuilt as a shared read-write-many cache backed by object
  storage, with file locking, checksum verification, and atomic replace —
  7B-parameter cold start ~40s → ~8s
- Multi-tenant inference gateway: Envoy ext_authz resolving tenant identity
  and SLA tier, SpiceDB fine-grained authorization (schema owned in
  Project 3), short-lived credentials
- GPU fleet instrumentation: NVIDIA DCGM, eBPF probes, OpenTelemetry →
  Prometheus / Grafana
- Metrics driving capacity planning, alerting, and incident response

## What to build

1. **Shared model cache**
   - RWX cache layer backed by S3-compatible object storage
   - File locking for concurrent readers/writers, checksum verification on
     read, atomic replace so no reader ever sees a partial model
   - Cache admission and eviction policy, with hit-rate metrics
2. **Cold-start measurement + optimization**
   - Cold start broken into phases: image pull, weight fetch, weight load,
     runtime warm-up — each timed separately
   - Optimizations applied per phase and re-measured (lazy/streamed loading,
     cache warming, snapshot reuse where applicable)
3. **Inference gateway**
   - Envoy with ext_authz: tenant identity and SLA tier resolved per request
   - SpiceDB permission check and short-lived credentials on the request path
   - Per-tenant rate and quota policy by SLA tier
4. **GPU observability stack**
   - DCGM exporter + eBPF probes, OpenTelemetry pipelines into
     Prometheus / Grafana
   - Dashboards: fleet utilization, per-endpoint latency, cache health,
     gateway auth latency
   - Alerts with runbooks; capacity report generated from the metrics

## Production-like scale targets

- A real 7B model (~4 GB at Q4 GGUF, ~14 GB fp16) for all cold-start numbers
- Concurrent cold starts racing for the same artifact; a corrupted cache
  entry injected on purpose
- Cold-start phases published separately — a single total number is not
  accepted as proof
- Gateway auth overhead measured against a stated p99 budget

## Design → production steps

- [ ] Design doc: cache consistency model, locking choice, atomic-replace
      mechanism, gateway request flow, telemetry cardinality budget
- [ ] Cache implementation with corruption and race tests passing
- [ ] Cold-start harness producing per-phase timings, before/after report
- [ ] Gateway deployed with ext_authz, SpiceDB checks, and quota policies
- [ ] Observability stack deployed; dashboards and alerts live
- [ ] One simulated incident run end-to-end using only the runbook
      (for example: cache corruption during peak cold starts)
- [ ] Capacity report written from the collected metrics

## Knowledge gaps this closes

- Where cold-start time actually goes, and which optimizations move which phase
- Distributed cache failure modes: partial writes, stale reads, lock contention
- Envoy ext_authz behaviour under auth-service slowness or outage
- Telemetry discipline: metric cardinality, eBPF overhead, alert quality
  vs alert volume

## Completion proof

- Repo with cache, gateway config, observability stack, and incident runbooks
- Cold-start report: per-phase before/after numbers at the stated model size
- This project's utilization dashboards are also the measurement proof for
  Project 1's ~40% → ~80% scheduling claim
