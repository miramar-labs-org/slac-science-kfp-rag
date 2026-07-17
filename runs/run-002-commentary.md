# run-002 — Commentary

Narrative observations from each monitoring tick — interpretation, concerns, notable trends.

---

### 07:10 PDT

Ingest cleanly rebuilt the Qdrant collection from the expanded 9-doc corpus (62 chunks, up
from 31) — confirms the `delete_collection`/`create_collection` pattern in `ingest_documents`
worked as expected and there's no cross-contamination from run-001's smaller corpus. Retrieval
is again perfect (recall@1/5/mrr/hit_rate all 1.0) over the full 36-question eval set, so the
larger corpus didn't introduce any retrieval ambiguity between topics. generation_eval just
started — this is the metric to watch, since the whole point of this run is to see whether the
denser context (more explicit numeric facts, redundant phrasing) pulls avg_answer_correctness
up from run-001's 3.2 to clear the 4.0 gate threshold.

### 07:19 PDT

generation_eval finished, and the result cuts against the hypothesis: avg_answer_correctness
came in at 3.0 (down from run-001's 3.2) and avg_fact_coverage at 0.50 (down from 0.55) — despite
a corpus with nearly double the word count and more explicit numeric phrasing. Two plausible
readings: (1) the eval set also grew from 20 to 36 questions, adding 4 brand-new topics
(FACET-II, Fermi-LAT, KIPAC, Nobel history) that the judge/generator may handle less well on
first exposure than the well-established original 5 — so this isn't a clean apples-to-apples
comparison of "same questions, denser context," it's "harder + broader question set, denser
context," and the difficulty increase may be outweighing the density benefit; or (2) the
densification approach itself (redundant phrasing, more numbers per doc) isn't the right lever
for this failure mode — worth checking a per-topic breakdown (old 5 vs new 4) once generation-eval
per-row data is available, rather than assuming the corpus-density theory is simply wrong.
faithfulness_eval and safety_eval are running now with the same transient judge-parse-error
pattern as run-001 (self-recovering, not fatal) — those numbers plus the gate result will tell
us whether this is a hard FAIL again or a borderline case.

### 07:24 PDT — FAIL

Gate failed again, on the same metric (`relevancy`), and the densified corpus made no
measurable difference — if anything, correctness ticked down (3.2→3.0) and fact coverage
too (0.55→0.50). Faithfulness/citation/safety are all still comfortably passing and basically
flat versus run-001, which is the useful signal here: the model isn't hallucinating or
citing poorly, it's specifically failing to reproduce exact figures precisely enough for the
correctness judge, and more redundant source text didn't fix that. My original diagnosis
(under-specification due to a thin corpus) looks wrong, or at least incomplete — the
qwen25-arc-kfp-rag precedent's fix (densify context) doesn't transfer cleanly to this project.
Confounding factor: the eval set grew from 20 to 36 rows in the same run as the corpus change,
so part of the drop could be the 16 new (harder, less-battle-tested) questions dragging the
average down rather than the existing 20 getting worse. Before trying another corpus edit,
the next diagnostic step should isolate old-vs-new questions in the per-row generation output,
or reconsider the generation/judging setup itself (self-judging with gpt-oss:20b, prompt
phrasing, or whether the judge's rubric is simply strict about exact-figure recall) as the
actual lever, rather than continuing to add more text to docs_src.