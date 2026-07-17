# Validation Status — slac-science-kfp-rag

**Type:** KFP v2 RAG pipeline
**Qdrant collection:** `slac-science-kfp-rag`
**Platform:** Kubeflow Pipelines on NVIDIA DGX Spark (GB10, 128 GB unified memory)
**Last updated:** 2026-07-17 (run-004)

---

## Current Status

| Component | Status |
|-----------|--------|
| `ingest_documents` | ✅ Implemented, validated |
| `retrieval_eval` | ✅ Implemented, validated (recall@5=1.0 every run) |
| `generation_eval` | ✅ Implemented, validated (judge `max_tokens` fix landed run-003) |
| `faithfulness_eval` | ✅ Implemented, validated (judge `max_tokens` fix landed run-003) |
| `safety_eval` | ✅ Implemented, validated |
| `deployment_gate` | ✅ Implemented, validated — **PASS on run-004** |

**Project has a fully passing gate result (run-004, gate_pass=1.0).** All 4 eval components
are implemented and validated end-to-end with real (non-fallback) judge output.

---

## Run Table

| Run | Purpose | Gate | recall@5 | faithfulness | relevancy | safety | Key Finding |
|-----|---------|------|----------|--------------|-----------|--------|-------------|
| run-001 | First run, 5-doc corpus | FAIL | 1.0 | 4.60 | 3.20 | 5.00 | Relevancy misses threshold — model under-specifies exact figures |
| run-002 | Corpus expanded 5→9 docs, eval 20→36 rows | FAIL | 1.0 | 4.61 | 3.00 | 5.00 | Corpus densification hypothesis rejected — relevancy got worse, not better |
| run-003 | Judge `max_tokens` fix (root cause: reasoning-model token truncation) | FAIL (infra) | 1.0 | — | 4.94 | — | Fix validated on generation_eval; run failed on unrelated PyPI pip-install timeout before gate could run |
| run-004 | Clean rerun, no code changes | **PASS** | 1.0 | 4.94 | 4.89 | 4.92 | First fully passing gate — fix confirmed stable on a second, completely clean run |

Full details: `runs/RUNS.md`, `runs/run-NNN.md`, `runs/run-NNN-commentary.md`.

---

## What Is Implemented

### Infrastructure (inherited from platform template)
- KFP v2 pipeline scaffold with all 6 stages wired
- MLflow run-per-stage tracking
- `ingest_documents` — chunking + embedding + Qdrant upsert (BAAI/bge-small-en-v1.5, CPU)
- `deployment_gate` — threshold checking for recall, faithfulness, relevancy, citation, safety
- `purge_kfp_mlflow.py`
- PVC mount: `hf-model-cache` at `/root/.cache/huggingface`
- Secret injection: `mlabs-api-keys` (OPENAI_API_KEY, HF_TOKEN, LANGCHAIN_API_KEY)

### Project-specific
- `config.yaml` — configured (llm.base_url → Ollama `gpt-oss:20b` on DGX, self-judging)
- `docs_src/` — 9 SLAC-topic documents (LCLS-II, SSRL, Rubin/LSST camera, DOE Genesis Mission, BaBar)
- `eval_dataset.jsonl` — 36 real Q&A rows, 4 per topic
- `notebook.ipynb` — all 4 USER CODE BLOCKs implemented per `WORKBOOK.md`; judge `max_tokens`
  raised (generation_eval 150→600, faithfulness_eval 300→700, safety_eval 150→500)

---

## What Is Still Pending

Nothing blocking. Optional future work:
- Independent (non-self) judge model, to remove the self-judging optimism trade-off noted in
  the original plan
- LangSmith tracing (`langsmith.enabled: true`) — not yet exercised

---

## Known Issues

None currently open.

> **Platform-level fixes** (bitsandbytes on Blackwell, trl 0.29 API, PIP_CONSTRAINT, etc.) are not
> applicable to this project — all pipeline steps are CPU-only and use no training libraries.

---

## Fixed Issues

### Judge `max_tokens` truncation (run-001 → run-003, fixed)

`gpt-oss:20b` is a reasoning model whose hidden chain-of-thought tokens count against the same
`max_tokens` budget as the visible JSON verdict returned to the judge calls in `generation_eval`,
`faithfulness_eval`, and `safety_eval`. The original budgets (150 / 300 / 150) were too small —
the model frequently exhausted the budget on reasoning before emitting any/complete JSON, hit
`finish_reason: "length"`, and silently fell back to hardcoded default scores
(`answer_correctness=3.0`, `fact_coverage=0.5`, `faithfulness=3.0`). This masked the pipeline's
true RAG quality behind what looked like a genuine relevancy gate failure in run-001/run-002.

**Fix:** raised judge `max_tokens` — `generation_eval` 150→600, `faithfulness_eval` 300→700,
`safety_eval` 150→500. Confirmed via run-003/run-004: judge errors dropped from 36/36 (run-002)
to 0/36, and relevancy jumped from 3.00 to 4.89–4.94, comfortably clearing the ≥4.0 threshold.

### Transient PyPI pip-install timeout (run-003, one-off, not recurring)

`faithfulness_eval`/`safety_eval` crashed at container startup on
`pip._vendor.urllib3.exceptions.ReadTimeoutError` against `files.pythonhosted.org` while
installing `openai`/`mlflow`. Confirmed as a transient network flake, not a code issue — did
not recur on run-004's clean rerun.
