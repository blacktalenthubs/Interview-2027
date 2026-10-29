# Apple — Model Lifecycle Pipeline & v3 Catalog (AI Core integration lead)
Status: DRAFT card

## Impact
- Model discovery → production became one automated pipeline with evaluation gates;
  nothing reaches the catalog without passing benchmarks and accuracy evals.
- The v3 catalog is the authoritative gating layer every downstream consumer trusts.

## The story in 2 minutes
Pipeline: HuggingFace scraping → security scans → artifact hydration → GGUF conversion →
quantization (Q4–Q8) → llama-bench benchmarks → evals (TruthfulQA, Winogrande, MMLU-Pro) →
production catalog, on Argo Workflows + Metaflow, per-step conda environments, idempotent
retries, parallel fan-out. Catalog v3: file collections, lifecycle management, staged
artifact storage, sandboxed remote-code validation. Cross-system integration owner across
Model Foundry / model-pipelines / ai-core / OB platform: per-branch sensor routing,
versioned API contracts, shared library.

## Hardest decisions / trade-offs
- Gate state recorded in the catalog vs per-system state (single source of truth).
- Create-only import for legacy models vs full re-processing (cost vs correctness).
- Sandboxed validation strictness vs model onboarding friction.

## Nasty questions
1. Eval mis-scored a multiple-choice task as perplexity — how did you catch it, and what
   does that say about trusting eval gates?
2. A model passes benchmarks but fails in production — where does the pipeline improve?
3. Idempotent retries across a 7-step pipeline — where is state anchored?
4. Why Argo AND Metaflow? Where does each earn its place?
5. Catalog is down — what breaks, what degrades gracefully?

## Gaps to fill
- [ ] Models in catalog (count), catalog consumers, curation throughput
- [ ] Inference servers/tenants served by the integrated systems
