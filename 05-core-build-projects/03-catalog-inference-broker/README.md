# Project 3 — Catalog + Inference Broker: Catalog v3, Redis Streams, SpiceDB

## Goal

Rebuild the authoritative data layer and the broker that tie the platform
together: the v3 model catalog that gates which models are available, and the
inference service that brokers inference across the platform — with
fine-grained authorization enforced on every call, and the cross-system
integration connecting foundry-style workflow triggering, curation execution,
the catalog/inference core, and the deployment/scheduling platform.

## Resume coverage

- Model Catalog API v3 — the authoritative data layer gating model availability:
  file collections, upstream model lifecycle management, staged artifact
  storage, remote code validation with sandboxed execution
- Inference service — server lifecycle endpoints, Redis Streams event
  architecture for server state transitions, health reporting from KServe
  runtimes back to the platform
- SpiceDB authorization — schema defined from scratch, relationship writes on
  resource creation (inference servers, models), permission enforcement on
  every API call
- Cross-system integration owner: per-branch sensor routing preventing
  cross-user pipeline fan-out, versioned API contracts preventing silent
  regressions, shared library used by every flow and service

## What to build

1. **Catalog API v3 (FastAPI + PostgreSQL)**
   - Data model: models, versions, file collections, lifecycle states
   - Staged artifact storage: nothing becomes available directly from staging
   - Sandboxed remote-code validation: unsafe model code cannot reach production
   - Versioned API contract with a compatibility check in CI
2. **Inference broker**
   - Server lifecycle endpoints (create, start, stop, health, delete)
   - Redis Streams consumer groups drive server state transitions —
     no polling loops
   - Health reporting ingested from KServe runtimes (Project 1)
3. **SpiceDB authorization layer**
   - Schema for tenants, models, inference servers, and roles
   - Relationship writes on resource creation, with a repair path when a
     write fails after the resource exists
   - Permission check on every API call in the catalog and the broker
4. **Integration layer**
   - Event/sensor routing scoped per branch/user so one user's pipeline
     never fans out to others
   - Shared client library used by the curation pipeline (Project 2),
     catalog, and broker

## Production-like scale targets

- ~10 tenants, hundreds of models and inference servers
- Duplicate and out-of-order state events delivered on purpose; final state
  must still be correct
- Negative authorization tests: cross-tenant access attempts must all fail
- Authorization latency measured and budgeted as part of API p99

## Design → production steps

- [ ] Design doc: catalog data model, lifecycle state machine, Streams event
      schema, SpiceDB schema, consistency choices and their trade-offs
- [ ] Catalog API with staged promotion flow and sandboxed validation service
- [ ] Broker with Streams consumer groups, dead-letter handling, state repair job
- [ ] SpiceDB integration with relationship-write repair and audit logging
- [ ] Shared library + contract tests between all three systems
- [ ] Chaos tests: duplicate events, out-of-order events, SpiceDB outage,
      Streams consumer crash mid-transition
- [ ] Runbook: stuck server state, authorization drift, catalog/staging mismatch

## Knowledge gaps this closes

- Zanzibar-style authorization: schema design, eventual consistency
  (zedtokens), check latency vs correctness trade-offs
- Event-driven state machines: at-least-once delivery, idempotent consumers,
  state repair as a first-class feature
- API versioning across systems owned by different teams without silent
  regressions

## Completion proof

- Repo with catalog, broker, auth schema, shared library, and contract tests
- End-to-end demo: model curated by Project 2 → promoted in catalog →
  served through the broker → called by an authorized tenant, and rejected
  for an unauthorized one
- Chaos test results published in the project README
