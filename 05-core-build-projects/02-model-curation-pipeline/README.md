# Project 2 — Model Curation Pipeline: Ingestion → Quantization → Eval Gate → Promotion

## Goal

Rebuild the fully automated end-to-end model curation pipeline: from a
HuggingFace model appearing upstream to a benchmarked, evaluated, promoted
model in the production catalog — with the evaluation harnesses acting as a
hard gate on publication.

## Resume coverage

- HuggingFace metadata scraping, security scans, artifact hydration
- GGUF conversion and quantization to multiple precision levels (Q4–Q8)
- Inference benchmarking with llama-bench
- Accuracy evaluation (TruthfulQA, Winogrande, MMLU-Pro) gating publication
- Built on Argo Workflows and Metaflow, per-step conda environments,
  idempotent retries, parallel fan-out across quantization targets
- The defect found and fixed in the original work: a multiple-choice
  evaluation mis-scored as a perplexity task

## What to build

1. **Ingestion** — HuggingFace metadata scraping, license and file inventory,
   security scans on artifacts and remote code before anything is trusted
2. **Hydration + conversion** — staged artifact download with checksums,
   GGUF conversion
3. **Quantization fan-out** — parallel branches per precision level (Q4–Q8),
   each in its own conda environment
4. **Benchmark + evaluation gate**
   - llama-bench performance numbers per quantization target
   - Accuracy evals with the scoring method matched to the task type
     (multiple-choice scored as multiple-choice, never as perplexity)
   - A model promotes only if every gate passes; failures stay visible
5. **Promotion** — passing artifacts and metadata handed to the catalog
   (Project 3 consumes this interface)

## Production-like scale targets

- 50–100 models curated end-to-end
- Failure injection at every step: corrupt artifact, failed scan, eval
  regression, killed worker mid-conversion
- Re-running any model produces the same artifacts and the same verdict
  (idempotency proven, not assumed)

## Design → production steps

- [ ] Design doc: pipeline DAG, artifact lineage model, retry/idempotency rules, gate policy
- [ ] Argo + Metaflow implementation with per-step conda environments
- [ ] Eval harness with task-type scoring tests — including a regression test
      that recreates the multiple-choice-as-perplexity mis-scoring and catches it
- [ ] Lineage record per model: source revision, checksums, scan results,
      benchmark and eval numbers per quantization level
- [ ] Failure-injection test suite passing
- [ ] Pipeline dashboards: step duration, retry rate, gate pass/fail rate
- [ ] Runbook: stuck pipeline, bad upstream model, gate dispute

## Knowledge gaps this closes

- Evaluation methodology: scoring per task type, eval variance, what a gate
  threshold should actually be
- Workflow reliability: idempotent step design, artifact passing vs reference
  passing, retry storms
- Quantization trade-offs: size vs accuracy per precision level, measured
  per model family instead of quoted from general knowledge

## Completion proof

- Repo with pipeline, eval harness, and lineage store
- A curated set of models with full lineage and gate history
- Written analysis of quantization accuracy/performance trade-offs from the
  project's own measurements
