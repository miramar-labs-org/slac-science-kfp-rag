# run-001 — Commentary

Narrative observations from each monitoring tick — interpretation, concerns, notable trends.

---

### 06:39 PDT

First-ever run for this project. `ingest_documents` just started — this step embeds the 5
SLAC docs with `BAAI/bge-small-en-v1.5` (CPU) and upserts to a fresh Qdrant collection
`slac-science-kfp-rag`. Nothing to judge yet; watching for it to reach `retrieval_eval` next,
which will give the first real signal on whether the doc corpus and eval questions are
well-aligned (gold_doc_ids match doc_id stems by construction, so recall should be high absent
a bug). The real unknown is generation/faithfulness/safety, since this is the first project run
self-judging with `gpt-oss:20b` for both roles.

### 06:44 PDT

Retrieval came back perfect (recall@1/5/mrr/hit_rate all 1.0) — no surprise given the eval set's
gold_doc_ids are constructed to match doc stems, but it confirms the ingest→chunk→embed→Qdrant
path is wired correctly with no off-by-one or collection mismatch bugs. Generation is the more
interesting result: avg_answer_correctness 3.2/5 and avg_fact_coverage 0.55 — noticeably softer
than the qwen25-arc-kfp-rag precedent's numbers. Worth watching whether this is a genuine
model/prompt weakness (gpt-oss:20b being less careful about extracting exact facts like dollar
figures and dates) or an eval-question issue (several of the 20 questions ask for precise
numbers — "$293.76 million", "32 million", "37 cryogenic modules" — which are easy for a judge
to mark wrong on units/rounding even when substantively correct). faithfulness_eval and
safety_eval are now running in parallel; faithfulness will be the real tell since it's judging
groundedness against retrieved context rather than exact-fact recall.

### 06:49 PDT — FAIL

First-run verdict: 5 of 6 gate metrics passed cleanly — retrieval (1.0), faithfulness (4.6),
citation_coverage (0.9), unsupported_claim_rate (0.0), and safety_score (5.0) all comfortably
clear their thresholds. This is a strong signal that the core RAG plumbing (ingest → chunk →
embed → Qdrant → retrieve → cite) and both judge prompts are correctly wired for a first attempt
— nothing here suggests a bug. The lone failure is `relevancy` (avg_answer_correctness) at 3.2
against a 4.0 threshold, paired with a soft `avg_fact_coverage` of 0.55. Given faithfulness and
citation_coverage are both high, the model isn't hallucinating or ignoring context — it's
under-specifying exact figures the eval's `required_facts` expect verbatim (dollar amounts, node
counts, dates). The precedent project (qwen25-arc-kfp-rag) hit an analogous correctness gap in
its early runs (run-003: 2.85, run-004: 3.60) and closed it by adding a denser paragraph-style
context document rather than changing the model or prompt — the same lever likely applies here:
either tighten the RAG_SYSTEM prompt to demand exact figures verbatim from context, or enrich
docs_src/ with more explicit fact statements near the numbers eval questions ask about. This is a
gate FAIL, but a shallow, well-understood one — not a re-architecture.
