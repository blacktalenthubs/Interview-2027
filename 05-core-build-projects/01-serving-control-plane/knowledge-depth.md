# Knowledge Depth — Serving Control Plane

Project 1 of the Core Build Projects. This document is the **study gate**:
it covers every concept, mechanism, architecture, and failure mode behind the
serving control plane at the depth needed to design it yourself. Do not move
to the design doc until the self-check at the end (§17) passes honestly.

Scope of this project: the **control plane** for model serving — the
Kubernetes operator and its `InferenceEndpoint` custom resource, the migration
to KServe, and admission-time GPU scheduling (MIG / time-slicing / dedicated).
Model storage, cold-start internals, and the gateway are Project 4; this
document touches them only where the control plane depends on them.

How to use it:

- Read in order. Sections 2–5 are the Kubernetes and KServe foundation,
  6–9 are the GPU and scheduling core, 10–16 are production behaviour.
- Every section ends in things you must be able to explain out loud, in your
  own words, without notes. That — not having read it — is the gate.
- Examples use the Current Employer platform shape from the resume: internal
  ML teams deploying models with no hand-written manifests, SLA tiers, GPU
  utilization raised from ~40% to ~80% by admission-time placement.

---

## 1. The big picture: what a serving control plane is

### 1.1 Control plane vs data plane

- **Data plane** = the components that serve a prediction request: the model
  server process (for example vLLM), the GPU, the proxy in front of it.
  Its metrics are latency (TTFT, time per token), throughput, error rate.
- **Control plane** = everything that decides what the data plane should look
  like: which model, on which node, on what fraction of a GPU, with how many
  replicas, reachable at which address, scaled by which policy. Its metrics
  are correctness and convergence speed: does reality match intent, and how
  fast after intent changes?

The operator you build is a control plane. KServe is a control plane with a
standard data plane contract. The admission webhook is a control-plane
decision point that runs synchronously in the API request path.

Confusing the two planes is the most common design error: putting request-path
work (per-token logic, per-request auth) into the operator, or putting
placement policy into the model server. Keep them separate.

### 1.2 The lifecycle this project owns

```
Team submits InferenceEndpoint (intent)
  → admission webhook validates + stamps placement (MIG / time-slice / dedicated)
  → operator (or KServe controller) reconciles intent into:
      Deployment, Service, autoscaler, RBAC, Ingress
  → kube-scheduler places pods on GPU nodes
  → kubelet + device plugin assign actual GPU devices / MIG instances
  → model server loads weights, becomes Ready
  → traffic flows through Service / Ingress
  → autoscaler changes replica count as load changes
  → team edits or deletes the InferenceEndpoint → reconcile again / teardown
```

Every arrow is a place where the system can be slow, wrong, or stuck — and
each one is covered below.

### 1.3 Why a platform at all — the problem being solved

Without a platform, every ML team writes its own manifests: inconsistent
resource requests, whole GPUs held by tiny models, no standard health checks,
no standard metrics, upgrades done by hand. The measured symptom on the
Current Employer platform was **~40% GPU utilization**: GPUs allocated, not
used. The platform's job is to make the right thing automatic: standard shape,
standard observability, and placement that matches the workload to the
hardware. The utilization gain (→ ~80%) came from placement and sharing, not
from buying different hardware.

---

## 2. Kubernetes foundations a platform builder actually needs

### 2.1 The declarative model

- You submit **desired state** (a resource object) to the API server. You
  never issue commands like "start a container on node 7".
- **etcd** is the only store: it holds desired state and observed state
  (status). Every component works off etcd through the API server.
- **Controllers** run reconcile loops: read desired state, read actual state,
  act to close the gap, repeat. This is *level-triggered* (driven by the
  current state) not *edge-triggered* (driven by events). A missed event is
  fine; the next sync sees the state. This is why Kubernetes systems are
  self-healing, and why your operator must be written the same way.
- Consequence: every action your controller takes must be **idempotent** —
  safe to run twice, safe to run after a crash mid-way, safe to run when
  someone else already did half of it.

### 2.2 The components, in request order

| Component | Job | What you must know |
|---|---|---|
| kube-apiserver | Authn/authz, admission, validation, persists to etcd | It is the *only* path to etcd; admission webhooks run inside this request path |
| etcd | Stores all objects | Watch mechanism powers every controller; large objects and watch storms are real limits |
| kube-scheduler | Assigns pods to nodes (binding) | Filter → Score → Bind pipeline, §9 |
| kube-controller-manager | Built-in controllers (Deployment, ReplicaSet, Node, …) | Your CRD gets the same treatment only if you write the controller |
| kubelet | Runs pods on a node | Talks to the container runtime (CRI) and to device plugins for GPUs |
| Device plugin | Advertises extended resources (`nvidia.com/gpu`, MIG resources) | Allocation happens at pod start on the node, not in the scheduler |

### 2.3 Resource anatomy

- **metadata**: name, namespace, labels (for selection), annotations (for
  non-identifying data — your webhook stamps placement decisions here),
  ownerReferences (garbage collection + "who manages this"), finalizers
  (block deletion until cleanup runs), resourceVersion (optimistic
  concurrency), generation (bumped on spec change).
- **spec**: desired state. **status**: observed state, written by the
  controller, never by the user. The `status.observedGeneration` field tells
  consumers whether status reflects the latest spec — a detail interviewers
  probe and many real systems get wrong.
- **Optimistic concurrency**: two writers updating the same object conflict on
  resourceVersion; the loser re-reads and retries. Controllers that ignore
  this corrupt state under load.

### 2.4 ReplicaSets and Deployments — what your operator stands on

- A Deployment owns ReplicaSets; a ReplicaSet owns Pods. Rolling update =
  new ReplicaSet scaled up while the old scales down, bounded by
  `maxSurge` / `maxUnavailable`.
- Readiness probes gate traffic: a pod that has not loaded its model must
  not receive requests. For GPU serving, readiness = weights loaded and the
  server answering its health endpoint — not "container started".
- PodDisruptionBudgets protect voluntary disruptions (node drains during
  upgrades) from taking every replica down at once.

**Explain-it gate (§2):** why level-triggered reconciliation beats reacting
to events; what breaks if status is written by users; why a container being
"Running" says nothing about a model being ready to serve.

---

## 3. CRDs and Operators, in depth

### 3.1 Custom Resource Definitions

- A CRD extends the API server with a new type. You define: group/version
  (`serving.example.io/v1`), kind (`InferenceEndpoint`), scope (Namespaced),
  schema (OpenAPI v3 structural schema — the API server validates against it),
  and subresources.
- **Subresources that matter:**
  - `/status` — spec and status are written through separate endpoints, so
    RBAC and update permissions separate cleanly.
  - `/scale` — exposes replicas through the standard scale API, which is
    what lets HPA autoscale a custom resource without knowing its internals.
- **Versioning:** v1alpha1 → v1beta1 → v1 is a contract evolution, not a
  rename. Storage version vs served versions, and conversion (webhook or
  None strategy) decide whether old objects survive an upgrade. Breaking a
  served version in production breaks every client pinned to it.
- **CEL validation** and defaulting: push validation into the schema where
  possible; reserve webhook validation for cross-field and cross-resource
  rules (for example "SLA tier gold requires a dedicated-capable node pool").

### 3.2 The Operator pattern

An operator = CRD + controller + domain knowledge encoded in software (the
runbook a human would follow, automated). The pattern's value is not "a
controller for a custom type" — it is encoding operational knowledge:
placement rules, upgrade order, backup/restore steps, failure recovery.

### 3.3 The reconcile loop, step by step

```
watch / informer cache delivers key (namespace/name)
  → workqueue (deduplicated, rate-limited per key)
  → get the InferenceEndpoint; if deleted → cleanup via finalizer path
  → compute desired child objects from spec (+ placement annotation)
  → compare with actual children (create / update / leave alone)
  → act through the API server (server-side apply or update)
  → write status: conditions, observedGeneration, endpoint address
  → on error: requeue with exponential backoff; on success: forget the key
```

Key mechanisms to know cold:

- **Informers / list-watch / shared cache:** controllers do not poll the API
  server per reconcile; they watch it and read from a local cache. Reconcile
  must tolerate a *stale* cache — another reason actions must be idempotent
  and conflicts retried.
- **Workqueue rate limiting:** a hot-looping error can hammer the API server;
  per-item exponential backoff (and a cap) is the standard defence.
- **Owner references + garbage collection:** children created with the CR as
  owner are deleted automatically when the CR is deleted, and the controller
  can watch children and re-reconcile the parent (a deleted Deployment gets
  recreated because the parent notices).
- **Finalizers:** when deletion must do real work first (deregister from a
  gateway, drain traffic, delete external resources), a finalizer keeps the
  object in `Terminating` until your controller finishes cleanup and removes
  it. Failure mode to know: a finalizer whose controller is dead = an object
  that can never be deleted. Every finalizer needs an escape procedure in the
  runbook.
- **Leader election:** run multiple operator replicas for HA, but only the
  leader reconciles (lease-based). Without it, two replicas fight over the
  same children and flap.
- **Status conditions:** use the standard shape (Type / Status / Reason /
  Message / LastTransitionTime) — `Ready`, `Progressing`, `Degraded` — so
  tooling and humans read every resource the same way.

### 3.4 kopf vs Kubebuilder / controller-runtime (Go)

- **kopf (Python):** handlers decorated per event (`@kopf.on.create`,
  `on.update`, `on.field`, `on.resume`), built-in finalizer and status
  handling, fast to build, matches a Python platform team. Trade-offs: Python
  performance in the watch loop is rarely the bottleneck (the API server is),
  but typed client ecosystem and envtest tooling are stronger in Go.
- **controller-runtime (Go):** the ecosystem standard (KServe itself is built
  on it); stronger for very large fleets and for contributing upstream.
- The choice must be defensible: team language, existing shared libraries,
  and who maintains it at 3 a.m. — not fashion.

### 3.5 Testing an operator

- **Unit:** reconcile as a pure-ish function — spec in, actions out.
- **envtest:** a real API server + etcd, no kubelet — validates schemas,
  admission, status subresource behaviour, and watch mechanics.
- **End-to-end:** a kind/k3d cluster with fake GPU nodes for placement tests.
- **Chaos:** kill the operator mid-reconcile; delete a child by hand and
  verify recreation; submit a spec edit during an upgrade.

**Explain-it gate (§3):** walk the reconcile loop from a `kubectl apply` to a
Running pod; explain what a finalizer deadlock looks like and how you recover;
explain why two operator replicas without leader election are worse than one.

---

## 4. The InferenceEndpoint abstraction

### 4.1 What a good platform API hides and what it must not hide

Hide: manifest boilerplate, RBAC objects, ingress wiring, probe definitions,
metric annotations, placement mechanics.
Do not hide: model identity and version, resource size, SLA tier, replica
bounds, traffic entry point. If users cannot express a real requirement in
the CRD, they will bypass the platform — and bypasses are how the 40%
utilization world returns.

### 4.2 Example spec (illustrative, this project's design target)

```yaml
apiVersion: serving.example.io/v1
kind: InferenceEndpoint
metadata:
  name: ranker-7b
  namespace: team-search
spec:
  model:
    uri: s3://models/ranker/7b/v14        # resolved via the catalog (Project 3)
    format: gguf
    quantization: q8
  runtime: vllm                            # or llamacpp, triton, tgi
  slaTier: gold                            # gold | standard | batch
  resources:
    gpuClass: auto                         # auto | mig | timeslice | dedicated
    minReplicas: 2
    maxReplicas: 8
  traffic:
    canary: { enabled: true, stepPercent: 10 }
status:
  conditions: [...]
  address: https://ranker-7b.internal/...
  placement: { mode: mig, profile: 3g.20gb }
  observedGeneration: 4
```

### 4.3 Derived resources — the operator's output contract

| Child | Why it exists | Owned decisions |
|---|---|---|
| Deployment | Runs the model server pods | Image, args, probes, resources (GPU request set by placement) |
| Service | Stable in-cluster address | Port, session affinity (usually none) |
| HPA / KPA config | Replica scaling | Metric and bounds from spec |
| RBAC (ServiceAccount, Role) | Pod identity, least privilege | Only what the pod needs (model read, metrics) |
| Ingress / HTTPRoute | External traffic entry | Host, TLS, canary weights |
| PodDisruptionBudget | Survive node maintenance | minAvailable from SLA tier |

### 4.4 Lifecycle states

`Pending → Provisioning (placement + scheduling) → Loading (weights) →
Ready → (Updating during rollout) → Degraded (replicas below intent) →
Terminating (finalizer cleanup)`. Each state must be observable from status
conditions alone — a user should never need to read operator logs to know
where their endpoint is stuck.

**Explain-it gate (§4):** given a new requirement ("this model needs two GPUs
and a warm standby"), decide whether it belongs in the CRD, and defend the
decision; explain which field changes trigger a rolling restart vs an
in-place update.

---

## 5. KServe, in depth

### 5.1 What KServe is

KServe is a standard control plane for model serving on Kubernetes: one CRD
family (`InferenceService` and friends), many runtimes, one inference
protocol. Building your own operator and then migrating to KServe — the
Current Employer path — is a common platform lifecycle: custom first (speed,
fit), standard later (ecosystem, maintenance, hiring).

### 5.2 Core resources

- **InferenceService** — the user-facing object. Three components:
  - **Predictor** (required): the model server itself.
  - **Transformer** (optional): pre/post-processing in the request path.
  - **Explainer** (optional): model explanations.
- **ServingRuntime / ClusterServingRuntime** — reusable runtime definitions
  (image, protocol, supported formats). vLLM, TGI, Triton, TorchServe,
  sklearn and others ship as runtimes; teams add their own.
- **TrainedModel** — a model version pointing at an InferenceService, used
  for multi-model and canary-by-model patterns.
- **InferenceGraph** — routing across services: ensembles, switches,
  splitters (A/B at the graph level).

### 5.3 What KServe creates per InferenceService

Depending on deployment mode:

- **Storage initializer** (init container): downloads model artifacts from
  object storage / PVC / registry into the pod before the server starts.
  This is a cold-start phase (Project 4 owns its optimization).
- **Agent** (sidecar): watches model storage and handles model load/unload
  in multi-model mode; reports model readiness.
- **Queue-proxy** (sidecar, serverless mode): measures concurrency, buffers
  during cold start, reports metrics to the autoscaler, enforces timeouts.
- The predictor container, a Service, and either a Knative Service
  (serverless mode) or a Deployment + HPA (raw mode).

### 5.4 Deployment modes — the decision that shapes everything

| | Serverless (Knative) | RawDeployment |
|---|---|---|
| Scaling | KPA autoscaler, scale-to-zero supported | HPA, min replicas stay warm |
| Cold start | Pays it on first request after zero | No zero-state cold start |
| Traffic | Knative routing, revision-based canary | Standard Ingress/Service routing |
| Best for | Spiky, low-duty-cycle, many small models | Latency-critical, steady production models |
| Cost of the choice | Queue-proxy + activator complexity | Idle GPUs still allocated |

Knowing *which mode a workload belongs in and why* matters more than knowing
either mode's flags.

### 5.5 The Open Inference Protocol (V2)

KServe standardizes on a REST/gRPC protocol (model metadata, health,
infer with typed tensors; LLM runtimes add OpenAI-compatible endpoints).
Why it matters for a platform: clients, transformers, and monitoring speak
one contract regardless of runtime, which is what makes a migration under
live traffic testable — the same client requests must succeed against the
old path and the new path.

### 5.6 Autoscaling in KServe

- **KPA (Knative Pod Autoscaler):** scales on concurrency or RPS measured by
  queue-proxy; supports scale-to-zero after an idle window, with a stable
  window to avoid flapping.
- **HPA (raw mode):** scales on CPU/GPU or custom metrics via metrics APIs.
- **LLM-specific wrinkle:** GPU memory (KV cache occupancy) and queue depth
  are often better scaling signals than request count for generative models;
  a request is not a uniform unit of work when prompts vary 100× in length.

### 5.7 Migration: custom operator → KServe, without downtime

The pattern, in order:

1. **Equivalence first:** define the contract tests — same model, same
   requests, both paths must return equivalent results within tolerance.
2. **Dual-run:** both control planes manage their own copy of the endpoint;
   the legacy path keeps production traffic.
3. **Shadow / mirror:** duplicate live traffic to KServe, compare outputs
   and latency, discard responses.
4. **Canary:** shift a small traffic percentage (Service/Ingress weights or
   InferenceGraph splitter); watch error rate and latency per path.
5. **Cutover per endpoint:** move endpoints in waves ordered by criticality
   (batch → standard → gold), never big-bang.
6. **Rollback:** traffic weights move back; both paths stay runnable until
   the migration window closes. "Both paths running side-by-side with no
   serving interruption" is the resume claim — this sequence is its proof.

Failure modes specific to migration: two controllers fighting over the same
child objects (prevent by separate namespaces/ownership), divergent defaults
(readiness timing, timeouts) causing behaviour differences users notice,
and metrics that are not comparable across paths — fix the dashboards
*before* the canary, not during the incident.

**Explain-it gate (§5):** draw what KServe creates for one InferenceService
in both modes; explain what the queue-proxy and agent each do; sequence a
no-downtime migration and name its rollback point at every step.

---

## 6. GPU fundamentals for serving

### 6.1 The hardware model you need

- A GPU is many **SMs** (streaming multiprocessors) executing kernels, fed
  by **HBM** (high-bandwidth memory) on the device. Two resources matter for
  inference: compute (SM throughput) and memory (capacity + bandwidth).
- **Inference is usually memory-bound, not compute-bound**, especially LLM
  decoding: each generated token requires reading the model weights (and the
  KV cache) from HBM. Bandwidth, not FLOPS, is the usual ceiling.
- Host ↔ device transfer runs over PCIe (or NVLink between GPUs). Loading
  weights crosses that path once at startup (cold start) — after that, a
  well-built server keeps everything resident on device.

### 6.2 Memory math — do it by hand

Model weight memory ≈ parameters × bytes per parameter:

| Precision | Bytes/param | 7B model | 70B model |
|---|---|---|---|
| FP16 / BF16 | 2 | ~14 GB | ~140 GB (multi-GPU) |
| INT8 | 1 | ~7 GB | ~70 GB |
| INT4 (Q4 GGUF) | ~0.5 | ~3.5–4 GB | ~35–40 GB |

Plus, per running model:

- **KV cache** (LLMs): grows with context length × concurrent sequences.
  Roughly: 2 (K and V) × layers × hidden size × context tokens × bytes.
  For a 7B model at 8k context this is on the order of 0.5 MB per token
  across the batch dimension in FP16 — in practice, several GB that must be
  budgeted or the server OOMs mid-traffic.
- **Runtime + CUDA context + activations:** budget 1–2 GB overhead minimum.

This math is the input to placement: "does this model fit a 20 GB MIG slice"
is a calculation, not a guess. Getting it wrong in the optimistic direction
is the classic cause of GPU OOM in production.

### 6.3 Batching and why utilization is not a single number

- **Static batching:** requests grouped in fixed batches; simple, wasteful.
- **Continuous batching** (vLLM and peers): sequences join and leave the
  batch per token step; this is the main reason modern LLM serving reaches
  high SM utilization at real-world request patterns.
- **PagedAttention** (vLLM): KV cache stored in fixed-size blocks like OS
  virtual memory pages — removes the fragmentation that forced huge
  worst-case KV reservations. Know the analogy to OS paging cold; it is the
  standard follow-up question on vLLM.
- Therefore "GPU utilization" has at least three meanings: allocation (is a
  GPU reserved), SM activity (DCGM's `DCGM_FI_DEV_GPU_UTIL`), and memory
  occupancy. §9.6 defines which one the 40% → 80% claim uses (allocation
  efficiency backed by SM activity).

**Explain-it gate (§6):** size a 13B Q8 model with 4k context for one GPU of
40 GB, showing the KV budget; explain why decoding is memory-bandwidth bound;
explain PagedAttention using the OS paging analogy.

---

## 7. GPUs in Kubernetes

### 7.1 The stack (NVIDIA GPU Operator bundles most of it)

| Layer | Component | Job |
|---|---|---|
| Driver | NVIDIA driver on the node | Talks to the hardware; version couples to CUDA |
| Runtime | NVIDIA Container Toolkit | Lets containers use the GPU (CDI / runtime hook) |
| Advertisement | **Device plugin** | Registers `nvidia.com/gpu` (and MIG resources) with kubelet |
| Discovery | Node Feature Discovery / GFD | Labels nodes: GPU model, memory, MIG capability, driver version |
| Telemetry | DCGM exporter | Per-GPU and per-MIG-instance metrics (Project 4) |
| Policy | GPU Operator / time-slicing config | Configures sharing mode per node pool |

### 7.2 The resource model — and its core limitation

GPUs are **extended resources**: integer, non-overcommittable, requested in
whole units (`resources.limits.nvidia.com/gpu: 1`). Kubernetes has no native
concept of "half a GPU" or "10 GB of GPU memory". Every sharing mechanism in
§8 exists to work around this one limitation:

- MIG makes the hardware itself present as several smaller "GPUs".
- Time-slicing makes the device plugin advertise N fake GPUs for one real
  one, and the hardware time-shares them.
- A scheduler/admission layer decides which illusion each workload gets.

Also know: the **scheduler** only counts the extended resource. The actual
device assignment happens later, on the node, by kubelet + device plugin
(`Allocate` returns device IDs / MIG instance handles via environment and
mounts). If the plugin and the scheduler disagree about what is free, pods
fail at start with allocation errors — a failure mode in §11.

### 7.3 Node hygiene

- **Taints** keep non-GPU workloads off GPU nodes (`nvidia.com/gpu:NoSchedule`
  + toleration on serving pods only). GPU nodes are the most expensive real
  estate in the cluster; a stray CPU pod there is a capacity bug.
- **Node pools by GPU type:** scheduling, MIG configuration, and time-slicing
  policy are per-pool decisions, expressed as labels/taints the placement
  logic (§9) targets.

**Explain-it gate (§7):** trace a GPU request from pod spec to a specific
device on a node, naming every component that touches it; explain why
Kubernetes alone cannot share one GPU between two pods.

---

## 8. Sharing a GPU: dedicated, MIG, time-slicing, MPS

This section is the heart of Project 1. The admission decision is a choice
between these four modes, per workload.

### 8.1 Comparison

| Mode | Isolation | Memory protection | Perf predictability | Density | Best for |
|---|---|---|---|---|---|
| Dedicated | Full GPU to one workload | Total | Highest | Lowest | Large models, gold SLA, multi-GPU |
| MIG | Hardware partitions (SMs + memory slices) | Hardware-enforced | High — partitions are fixed | High | Small/medium models needing real isolation |
| Time-slicing | None — contexts interleave on the SMs | None — any tenant can OOM the device | Low — neighbours affect latency | Highest | Dev, batch, best-effort tiers |
| MPS | Shared context, partial SM partitioning ( Volta+ ) | Partial (per-client memory limits possible) | Medium | High | Many small kernels under one administrative domain |

### 8.2 MIG (Multi-Instance GPU), in depth

- Available on A100 / H100 (and successors). One physical GPU is split into
  **GPU Instances (GIs)**, each optionally split into **Compute Instances
  (CIs)**. A MIG instance has dedicated SMs, dedicated L2/cache paths,
  dedicated memory controllers and a fixed memory size — it is a hardware
  partition, not a software quota. A fault in one instance does not take
  down the others (fault isolation is the enterprise selling point).
- **Profiles are fixed geometry.** On an A100-40GB, examples:
  `1g.5gb`, `2g.10gb`, `3g.20gb`, `4g.20gb`, `7g.40gb`. An A100-80GB / H100
  offers larger variants (for example `3g.40gb` on 80 GB parts). You cannot
  ask for "12 GB" — you pick the nearest profile, and the leftover geometry
  constrains what else fits. Instances must fit the GPU's placement rules
  (slices are positioned, not arbitrary), so valid combinations are a
  packing problem, not simple addition.
- In Kubernetes, MIG surfaces two ways:
  - **Mixed strategy:** the device plugin advertises typed resources like
    `nvidia.com/mig-3g.20gb: 1` — the scheduler sees profile types per node.
  - **Single strategy:** nodes are pre-partitioned uniformly and advertise
    plain `nvidia.com/gpu` counts of the slice.
- **Reconfiguration is disruptive:** changing MIG layout requires draining
  the GPU (no running workloads) — MIG mode toggling historically requires
  privileged operations and can require a reset. Consequence: MIG layout is
  a *node-pool-level, slow-changing* decision. A platform cannot repartition
  per request; it pre-partitions pools into a small set of layouts and lets
  admission choose among existing capacity.
- Memory math from §6.2 decides fit: a 7B FP16 model (~14 GB + KV + overhead)
  fits a `3g.20gb` slice but not a `2g.10gb` one; the same model at Q4 fits
  much smaller geometry. Placements must use the *quantized artifact's* real
  footprint, not the parameter count alone.

### 8.3 Time-slicing, in depth

- The device plugin is configured with `replicas: N`: it advertises N
  devices per physical GPU, and the CUDA runtime time-shares contexts
  (default scheduler round-robins at a time quantum).
- No memory isolation: each tenant sees the whole device memory; the sum of
  tenant footprints must stay under capacity or *someone* gets OOM-killed at
  allocation time — possibly a different tenant than the one that grew.
- No compute isolation: a neighbour's kernel bursts are your latency jitter.
- Therefore: time-slicing is correct for dev / batch / internal best-effort,
  and a production incident waiting to happen for gold-SLA multi-tenant
  serving. Being able to say *where the line is* is the staff-level answer.

### 8.4 MPS (Multi-Process Service), briefly

A daemon lets multiple processes share one GPU context with (on modern
architectures) per-client SM percentage limits (`CUDA_MPS_ACTIVE_THREAD_PERCENTAGE`)
and per-client memory caps. Better utilisation for many small kernels than
separate contexts, weaker operational story in Kubernetes (extra daemon,
shared fate) — in practice most platforms choose MIG or time-slicing and
keep MPS for specialised single-team nodes.

### 8.5 The decision, stated as a rule set (starting point for §9)

- Model does not fit any MIG profile, or SLA is gold with strict p99 →
  **dedicated**.
- Model fits a profile with KV headroom, tenant isolation required →
  **MIG**.
- Internal / dev / batch, tolerant of jitter and OOM-by-neighbour →
  **time-sliced**.
- Burst overflow: allow standard tier to spill to time-sliced capacity,
  never gold.

**Explain-it gate (§8):** given an A100-40GB partitioned as one `3g.20gb` +
two `1g.5gb`, decide which remaining workloads can be placed and why; explain
to a sceptical tenant why their time-sliced latency doubled overnight;
explain why MIG cannot be reconfigured on the fly per request.

---

## 9. Scheduling and placement, in depth

There are **three different decision layers**. Keeping them separate is the
core architectural insight of this project.

| Layer | When | Decides | Mechanism in this project |
|---|---|---|---|
| 1. Admission | Pod/CR creation, synchronous, milliseconds | *Mode*: MIG vs time-slice vs dedicated, and which pool | Mutating admission webhook |
| 2. Scheduling | Pod pending → bound | *Node*: which node with free capacity of that mode | kube-scheduler (stock) |
| 3. Allocation | Pod starting on the node | *Device*: which physical GPU / MIG instance | kubelet + device plugin |

### 9.1 Admission webhooks

- **Mutating** webhooks run before **validating** webhooks, before the object
  is persisted. The API server sends an `AdmissionReview`; your webhook
  returns allowed/denied plus (for mutating) a JSONPatch.
- In this project the mutating webhook: reads the endpoint's model size and
  SLA tier, computes the placement (§8.5), and stamps it: resource requests
  rewritten (`nvidia.com/mig-3g.20gb: 1` or a time-sliced replica), node-pool
  affinity added, decision recorded in an annotation for audit.
- **Design constraints that bite in production:**
  - *Latency budget:* the webhook sits in every create/update for its
    resources. Target single-digit ms p50, hard timeout in tens of ms.
    A slow webhook is a slow platform; a slow dependency (catalog lookup)
    must be cached, never called synchronously per admission.
  - *Failure policy:* `failurePolicy: Fail` = webhook down means no new
    endpoints (safe, blocks the platform); `Ignore` = endpoints admitted
    unplaced (unsafe defaults, leaks whole GPUs). There is no comfortable
    answer — gold fleets choose Fail plus a highly-available webhook;
    whatever you choose, it must be a written decision with a runbook.
  - *Idempotency:* webhooks can be reinvoked; a patch computed from a
    half-patched object must converge, not stack changes.
  - *Scope discipline:* match rules narrowly (only your CRD, only
    CREATE/UPDATE) or you will eventually break someone else's workload —
    including the webhook's own deployment.

### 9.2 kube-scheduler, enough to reason about placement outcomes

Pipeline per pod: **queueing → filtering (predicates) → scoring → binding**,
with extension points (PreFilter, Filter, Score, Reserve, Permit, PreBind,
Bind). Filters relevant to GPU serving:

- `NodeResourcesFit`: requested extended resources ≤ node's allocatable.
- Taints & tolerations; node affinity (pool labels); inter-pod
  anti-affinity (spread replicas across nodes so one node loss is not an
  outage).
- Scoring then prefers, by plugin weights, e.g. least-allocated (spread) or
  most-allocated (binpack). **For GPUs you usually want binpacking within a
  pool**: spread leaves half-empty GPUs everywhere — allocation
  fragmentation with good intentions.

What the stock scheduler does *not* do: gang scheduling (a multi-pod
inference group is scheduled pod-by-pod; partial placement can strand
GPUs), topology awareness beyond labels (NVLink domains need explicit
labelling/plugin work), and dynamic rebalancing (it never moves a running
pod to defragment — see §9.5).

### 9.3 The placement algorithm (worked)

Inputs: model footprint (from §6.2 math on the actual artifact), SLA tier,
tenant isolation requirement, current pool capacity by profile.

```
if footprint > largest MIG profile or sla == gold and strict-p99:
    mode = dedicated
elif fits(profile) with >= 20% KV headroom and isolation required:
    mode = mig; profile = smallest fitting profile
elif tier in (dev, batch) or burst-spill allowed:
    mode = timeslice; replicas-share = smallest share count that fits memory
else:
    mode = dedicated (safe default)
```

Worked examples:

- **7B Q8 (~8 GB + ~4 GB KV + overhead ≈ 13 GB), standard tier, shared
  fleet** → MIG `3g.20gb` (fits with headroom; `2g.10gb` does not).
- **7B FP16 (~14 GB + KV ≈ 18–20 GB), gold tier** → dedicated, despite the
  near-fit in `3g.20gb`: gold p99 cannot share fate with a repartition or a
  neighbour's driver fault story, and KV growth at long context would
  breach the slice.
- **1B embedding model (~2 GB), dev tier** → time-sliced; isolation would
  waste a MIG slice that a 7B needs.

### 9.4 SLA tiers as a platform contract

A tier is not a label; it is a bundle of enforceable settings:

| | Gold | Standard | Batch / dev |
|---|---|---|---|
| Placement | Dedicated | MIG (dedicated on overflow policy) | Time-sliced |
| PriorityClass | Highest; can preempt batch | Normal | Lowest; preemptible |
| Min replicas | ≥ 2 across nodes | ≥ 1 | 0 allowed (scale-to-zero ok) |
| PDB | minAvailable ≥ 1 | Optional | None |
| Node pool | Production GPU pool | Production GPU pool | Overflow / dev pool |

Priority + preemption is how the tiers behave under scarcity: when the fleet
is full and a gold endpoint arrives, batch pods are evicted (they chose that
contract) rather than the gold endpoint pending forever.

### 9.5 Fragmentation — the slow killer of utilization

- **Definition:** fleet-wide free GPU capacity exists, but not in the shape
  any pending workload needs: free MIG slices of the wrong profile, GPUs
  with 15 GB free when the next model needs 20 GB, one free GPU on each of
  four nodes when a job needs two on one node.
- **Causes:** mixed profiles per node, FIFO arrivals, long-lived small
  tenants pinning slices, spread-based scheduling (§9.2).
- **Mitigations, in order of increasing violence:** binpack scoring;
  standardizing a small set of MIG layouts per pool; over-provisioning the
  most-demanded profile; periodic defragmentation (drain a node, evict and
  reschedule its tenants in a maintenance window, repartition cleanly);
  backfill scheduling (fit small batch work into fragments gold cannot use).
- **Measuring it:** track *allocatable-but-unplaceable* capacity: simulate
  the pending queue against free capacity and report how much free capacity
  no queued workload could use. If you do not measure fragmentation, you
  will "discover" it as an outage during a launch.

### 9.6 Utilization — defining the headline metric honestly

- **Allocation utilization:** GPU-hours allocated to workloads ÷ GPU-hours
  in fleet. The pre-platform failure mode: a 1B model holding a whole
  A100 counts 100% allocated, ~5% used.
- **Activity utilization:** DCGM SM activity / memory throughput averaged
  over the window.
- The platform claim (~40% → ~80%) is an **allocation-efficiency** claim
  backed by activity sampling: after admission placement, small models hold
  slices instead of GPUs, so the same fleet serves roughly double the
  workload. Any honest report shows: fleet size, window, workload mix, and
  both metrics. Averages without a mix statement are how utilization claims
  get destroyed in review — and in interviews.

### 9.7 Advanced topics to know exist (name + one-line when-to-use)

- **Kueue:** Kubernetes-native job queueing with quotas and cohort sharing —
  for batch/fine-tuning fleets sharing capacity fairly.
- **Volcano / gang scheduling:** all-or-nothing scheduling for distributed
  training and multi-pod inference groups.
- **Multi-cluster / fleet scheduling:** placement across clusters by
  capacity and data locality — the next scale step after one fleet saturates.
- **Descheduler:** evicts pods by policy (duplicates, underutilized nodes)
  to enable rebalancing the stock scheduler never does on its own.

**Explain-it gate (§9):** a gold endpoint is Pending while the dashboard
shows 30% of the fleet free — give three distinct causes and how you'd check
each; defend `failurePolicy` for the admission webhook; compute whether the
fleet can take one more 7B FP16 gold endpoint, given a stated layout.

---

## 10. Autoscaling and traffic, control-plane side

- **HPA** scales on metrics via the scale subresource: CPU by default; GPU
  and request metrics require metrics adapters (Prometheus adapter, KEDA).
  Scaling a GPU workload is slow (pod start + weight load), so HPA configs
  need stabilization windows and lower scale-down rates than CPU services.
- **KPA (Knative)** scales on concurrency/RPS from queue-proxy and can scale
  to zero (§5.4). Choose per workload duty cycle, not per team preference.
- **The scale-up trap:** a new replica is not capacity until weights are
  loaded and, for LLMs, the KV allocator is warm. Readiness gates traffic,
  but autoscalers that count starting pods as capacity will shed load
  incorrectly. Scale *early* (predictive headroom via min replicas for gold)
  beats scaling fast.
- **Traffic management:** canary by weight at Ingress/HTTPRoute or
  InferenceGraph; header-based routing for internal test tenants; retries
  only for idempotent inference calls, with budgets — retry storms against a
  saturated GPU fleet make outages worse, not better.

**Explain-it gate (§10):** why GPU HPA needs different stabilization than
CPU HPA; when scale-to-zero is the wrong choice even for a low-traffic model.

---

## 11. Failure modes and blast radius

| Failure | Symptom | Blast radius | Mitigation / recovery |
|---|---|---|---|
| Operator down | Spec changes ignored; running pods unaffected | Control plane only | HA replicas + leader election; alert on reconcile lag |
| Webhook down (Fail) | All creates/updates rejected | Platform-wide writes | Webhook HA, tight SLO, runbook to temporarily relax policy |
| Webhook bug admits unplaced pod | Whole GPU consumed by small model | Capacity | Validating webhook as second gate; periodic placement audit job |
| API server / etcd stress | Slow everything, watch lag | Cluster-wide | Operator caching, backoff, list-watch hygiene |
| Device plugin mis-advertises | Pods fail allocation at start | Node pool | Node health checks, plugin config in GitOps, canary node upgrades |
| MIG layout drift (manual change) | Scheduler counts capacity that is not there | Node pool | MIG state reconciled by GPU operator; drift alerts |
| GPU hardware fault (Xid errors, ECC) | Pod crashes, node degrades | Node | DCGM health monitoring, node cordon automation, pod rescheduling |
| GPU OOM inside a time-sliced pod | Neighbour pod dies, not just the offender | All tenants on that GPU | Tier discipline (§8.3): no gold on time-sliced nodes |
| Node loss | All replicas on the node die | Endpoints without anti-affinity | Anti-affinity + PDBs + min replicas across nodes (gold) |
| KServe controller down (after migration) | Same shape as operator down | KServe-managed endpoints | Same HA pattern; legacy path unaffected — a migration benefit |
| Bad canary during migration | Elevated errors on shifted % | That endpoint's users | Automatic rollback on error budget; per-endpoint waves (§5.7) |
| Finalizer deadlock | Endpoint stuck in Terminating | One endpoint (+ user trust) | Finalizer timeout + documented force-clean procedure |

Rule that falls out of the table: **control-plane failures must degrade to
"no changes", never to "serving stops".** If an operator outage can take
down running models, the architecture is wrong.

---

## 12. Security and multi-tenancy essentials

- **The operator is a privileged target.** It typically holds
  create/update/delete on Deployments, Services, RBAC in tenant namespaces.
  Scope it per namespace where possible, audit its actions, and treat its
  image supply chain like production code at the highest tier — a compromised
  operator owns the fleet.
- **Tenant isolation layers** stack: namespace + RBAC (API), ResourceQuota
  (GPU counts per tenant — quotas work on extended resources), placement
  (MIG hardware isolation), network policy (traffic), and admission
  (validating webhook enforcing quota and tier rules at write time).
- **Model artifacts are code.** A model repo can carry executable code
  (trust_remote_code patterns). The control plane records artifact identity
  and checksum; sandboxing and scanning belong to Projects 2/3, but the
  serving layer must refuse artifacts whose identity does not match the
  catalog record.
- **Secrets:** model registry credentials via short-lived, workload-identity
  tokens — never static keys baked into images or CRD specs.

---

## 13. Observability of the control plane itself

Minimum signal set (dashboards/alerts are built in Project 4; the control
plane must *emit* these from day one):

- Reconcile: rate, error rate, duration histogram, queue depth, per-resource
  lag (time from spec change to status Ready).
- Admission: request rate, p50/p99 latency, deny rate by reason, placement
  decision distribution (MIG / time-slice / dedicated per tier).
- Scheduling outcomes: pending pod count and age by reason
  (insufficient MIG profile is a distinct reason, not generic
  "insufficient gpu").
- Fleet: allocation vs activity utilization (§9.6), fragmentation estimate,
  capacity by profile per pool.
- SLOs: e.g. "95% of endpoint creates reach Ready in < 5 min for cached
  models", "admission p99 < 50 ms". Control-plane SLOs are separate from
  serving SLOs, owned separately, alerted separately.

---

## 14. Trade-offs and economics worth being able to argue

- **Utilization vs latency:** every sharing step raises utilization and
  worsens tail latency or isolation. The tier system exists so this is a
  per-workload business decision, not a global one.
- **Utilization vs resilience:** an 80%-allocated fleet has 20% headroom for
  node failures and launch spikes. 95% allocation is a headline and an
  incident. Staff-level answer: state the headroom policy explicitly.
- **Custom operator vs KServe:** custom = fit and speed now, maintenance
  forever; standard = ecosystem and shared fixes, least-common-denominator
  abstractions. The migration on this resume is the honest lifecycle of
  that trade-off.
- **Admission decision vs scheduler plugin:** admission is simple,
  synchronous, and early — but cannot see real-time node state precisely;
  a scheduler plugin sees pending state but is harder to build and upgrade.
  Choosing admission + stock scheduler is defensible *if* you can name what
  you gave up (exact binpacking across the pending queue, gang semantics).
- **Cost framing:** GPU-hours are the unit cost. Placement improvements are
  worth (workloads served per GPU-hour) — report improvements in those
  terms to leadership, not raw percentages alone.

---

## 15. Worked example — one endpoint, end to end

`ranker-7b`, standard tier, Q8 artifact (from §4.2):

1. **T+0s** — Team applies the CR. Mutating webhook: footprint ≈ 13 GB,
   standard tier → MIG `3g.20gb`; patch sets
   `nvidia.com/mig-3g.20gb: 1`, pool affinity `gpu-pool-prod`, annotation
   records the decision and inputs. Validating webhook checks tenant quota.
2. **T+1s** — Object persisted; operator reconcile fires from the watch.
3. **T+2s** — Operator (server-side apply) creates Deployment (1 replica
   initially), Service, HPA, RBAC, HTTPRoute — all owned by the CR.
   Status: `Provisioning`.
4. **T+3s** — Scheduler filters nodes: needs a node advertising a free
   `mig-3g.20gb` resource in the prod pool, toleration matches the GPU
   taint; binpack scoring picks the fullest suitable node. Pod bound.
5. **T+5s** — Kubelet + device plugin allocate the MIG instance; storage
   initializer pulls weights into the model cache (Project 4); the server
   container starts. Status: `Loading`.
6. **T+~40s** (uncached) or **~8s** (cached) — Weights loaded, health
   endpoint green, readiness probe passes, pod enters Service endpoints.
   Status: `Ready`, `observedGeneration` = current generation.
7. **Steady state** — HPA watches request metrics; at sustained load it
   scales to 3 replicas, each repeating steps 4–6 (minus webhook; scaling
   is the Deployment's business, placement was stamped on the pod template).
8. **Edit** — Team bumps the model version. Generation increments; rolling
   update replaces pods one at a time under the PDB; canary policy routes
   10% to the new revision first (if enabled in spec).
9. **Delete** — CR deleted; finalizer drains the endpoint from the gateway
   and releases the catalog lease (Project 3), children are garbage-collected
   by owner reference, finalizer removed, object gone.

If you cannot narrate this timeline and name the owner of each step, the
design phase will be built on sand — go back to the section for that step.

---

## 16. Anti-patterns and interview traps

- **Polling the API server** from a controller instead of watch/informers.
- **Writing business logic into the webhook** that needs slow external calls.
- **Treating `nvidia.com/gpu: 1` as a size** — it is a count of whole devices;
  size lives in MIG profiles and memory math.
- **Quoting utilization without defining it** (§9.6).
- **Claiming MIG can be resized live** to fit the next request (§8.2).
- **Placing gold and batch on the same time-sliced nodes** and calling it
  multi-tenancy (§8.3).
- **One giant operator service account** across all tenant namespaces (§12).
- **Status written but `observedGeneration` ignored**, so clients act on
  stale readiness (§2.3).
- **Migration by replacement** (delete old, create new) sold as
  "no downtime" (§5.7).
- **Autoscaling on pod count alone** for LLM workloads (§5.6, §10).

---

## 17. The gate — self-check before design

Answer every item out loud or in writing, without the document. Any gap
sends you back to that section before design starts.

**Concepts (§1–2)**
- [ ] Control plane vs data plane, with two metrics for each
- [ ] Why reconciliation is level-triggered, and what that demands of my code
- [ ] spec vs status, generation vs resourceVersion vs observedGeneration

**Operators (§3–4)**
- [ ] Full reconcile loop, including workqueue backoff and leader election
- [ ] Finalizers: purpose, deadlock, recovery
- [ ] kopf vs controller-runtime trade-off, defended for a Python team
- [ ] The InferenceEndpoint spec: what I hide, what I expose, and why

**KServe (§5)**
- [ ] Everything created for one InferenceService, both deployment modes
- [ ] Queue-proxy and agent responsibilities
- [ ] The six-step no-downtime migration, with rollback at each step

**GPU + sharing (§6–8)**
- [ ] Memory sizing for a given model/precision/context, by hand
- [ ] MIG geometry and why reconfiguration is a pool-level decision
- [ ] Time-slicing's failure story, and which tiers may use it

**Scheduling (§9)**
- [ ] The three decision layers and what each one decides
- [ ] Webhook failure policy, defended with its blast radius
- [ ] Fragmentation: causes, measurement, four mitigations
- [ ] Utilization: the two definitions, and how the 40% → 80% claim is built

**Production behaviour (§10–14)**
- [ ] Five failure modes from §11 with mitigation, from memory
- [ ] Why control-plane outage must never stop serving
- [ ] The utilization-vs-resilience headroom argument

**Narration**
- [ ] The §15 timeline, end to end, naming each step's owner

When every box is honestly checked, the next document in this folder is the
design: `design.md` — architecture, API contracts, and build plan for
Project 1.
