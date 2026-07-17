# run-003 — Commentary

Narrative observations from each monitoring tick — interpretation, concerns, notable trends.

---

### 09:05 PDT

The judge-truncation fix is working exactly as diagnosed. In run-002, 36/36 generation_eval
rows hit "Judge error" and silently fell back to corr=3.0/fc=0.50 — that's why relevancy sat
at a suspiciously flat 3.0-3.2 across both prior runs regardless of corpus changes. With
max_tokens raised from 150 to 600, 33/33 rows processed so far have succeeded with real
scores: 31 at corr=5.0, 2 at corr=4.0, fact_coverage=1.00 across the board. If this holds for
the last 3 rows and faithfulness_eval (also fixed, 300->700) behaves the same way, this run
should clear the relevancy>=4.0 gate comfortably — the first real, non-fallback-masked
measurement of this pipeline's actual quality. Retrieval is unchanged (perfect, as expected).
Next: watch generation_eval's final average, then faithfulness_eval and safety_eval for the
same error-rate collapse, then the gate result.

### 09:10 PDT — FAILED (infra flake, fix validated)

generation_eval finished with avg_answer_correctness=4.94 and avg_fact_coverage=0.99 —
essentially perfect, and zero judge errors across all 36 rows. This is the confirmation the
whole diagnosis was chasing: the relevancy gate failures in run-001 and run-002 were never
about corpus thinness or density, they were the judge silently failing on every call and
falling back to a hardcoded 3.0/0.50. With the token budget fixed, the model's actual
correctness is excellent. Unfortunately faithfulness_eval and safety_eval both crashed before
producing any output — a PyPI read-timeout while pip-installing `openai`/`mlflow` into their
containers, completely unrelated to the RAG code or the fix. The gate never got to run since
it depends on both judge stages. This is a clean infra flake: the fix is proven, the pipeline
just needs a straightforward rerun to get faithfulness/safety/gate numbers. No code changes
needed before that rerun.
