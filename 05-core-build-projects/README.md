# Core Build Projects — Current Employer Role

Four production-like build projects that together cover the full Current Employer
role on the AI Infrastructure resume (v2): model serving and inference platform
leadership, plus the systems integration behind the end-to-end model lifecycle —
curation pipeline, v3 model catalog, and inference service connecting the model
foundry, curation pipelines, catalog, and deployment/scheduling platforms.

Goal of each project: take it from design to production-like — design doc,
versioned API contracts, working implementation, load and failure testing,
observability, runbook — so every resume claim can be defended with a built
system and measured numbers, and every knowledge gap behind the claim is closed.

## The four projects

| # | Project | Covers | Headline proof |
|---|---------|--------|----------------|
| 1 | [Serving Control Plane](01-serving-control-plane/README.md) | Kubernetes operator (InferenceEndpoint CRD), KServe migration, admission-time GPU scheduling (MIG / time-slicing / dedicated) | GPU utilization ~40% → ~80% under replayed workloads |
| 2 | [Model Curation Pipeline](02-model-curation-pipeline/README.md) | HuggingFace ingestion, security scans, hydration, GGUF conversion, Q4–Q8 quantization fan-out, llama-bench, accuracy evals gating promotion | 50–100 models curated end-to-end with zero unsafe or mis-scored promotions |
| 3 | [Catalog + Inference Broker](03-catalog-inference-broker/README.md) | Model Catalog API v3, staged artifact storage, sandboxed remote-code validation, Redis Streams inference service, SpiceDB authorization, cross-system integration | Multi-tenant catalog + broker with authorization enforced on every call |
| 4 | [Runtime Performance + Observability](04-runtime-performance-observability/README.md) | Shared RWX model cache on object storage, cold-start optimization, Envoy inference gateway, DCGM / eBPF / OpenTelemetry → Prometheus / Grafana | 7B model cold start ~40s → ~8s, measured by phase |

## Coverage map — resume skill → project

- KServe, vLLM, llama.cpp / llama-bench, GGUF quantization (Q4–Q8), model evaluation → Projects 1, 2
- Kubernetes operators & custom resources (kopf), MIG partitioning, GPU time-slicing, admission-based GPU scheduling → Project 1
- Argo Workflows, Metaflow, Helm, Terraform, Docker, GitOps, AWS (EKS, S3, EC2, EMR) → Projects 1, 2, 4
- Model catalog & lifecycle APIs, data modeling, data quality / validation / monitoring, HuggingFace ingestion, staged artifact storage, sandboxed remote-code validation → Projects 2, 3
- Redis Streams, server lifecycle and state transitions, KServe health reporting → Project 3
- Python, FastAPI, REST / gRPC, PostgreSQL, Redis → Projects 1, 3
- Envoy ext_authz, SpiceDB (Zanzibar-style authorization), OAuth 2.0, short-lived credentials → Projects 3, 4
- Prometheus, Grafana, OpenTelemetry, NVIDIA DCGM, eBPF, telemetry (metrics, logs, traces), capacity planning and incident response → Project 4
- Cross-system integration: versioned API contracts, shared library, per-branch sensor routing → Project 3 (consumes Projects 1, 2, 4)

Out of scope for these four (they belong to earlier roles on the resume, not the
Current Employer role): Kafka exactly-once streaming, Spark Structured Streaming,
and the Go / Java payment platforms.

## Build order

1. **Project 1** — serving foundation first.
2. **Project 4** — instrument and speed up Project 1 immediately; it also
   produces the measurement proof for Project 1's utilization claim.
3. **Project 2** — feed the serving platform real curated models.
4. **Project 3** — capstone: connects foundry-style triggering → curation
   (Project 2) → catalog → inference broker → serving (Projects 1, 4).

## What "production-like" means for every project

Each project folder tracks the same completion bar:

- [ ] Design doc with architecture, data model, failure modes, and trade-offs
- [ ] Versioned API contract (OpenAPI / protobuf) with a compatibility check in CI
- [ ] Working implementation deployed with Helm + GitOps
- [ ] Load test with published numbers, not estimates
- [ ] Failure-injection tests for the project's named failure modes
- [ ] Dashboards, alerts, and an on-call runbook
- [ ] README with measured results and an honest note on what is real vs simulated
      (for example: real MIG requires one MIG-capable cloud GPU node; the rest of
      the fleet can be simulated nodes)

## Status

- [ ] Project 1 — Serving Control Plane
- [ ] Project 2 — Model Curation Pipeline
- [ ] Project 3 — Catalog + Inference Broker
- [ ] Project 4 — Runtime Performance + Observability
