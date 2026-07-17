# RUNS

| Run | Date | Gate | Retrieval R@5 | Faithfulness | Relevancy | Citation Cov | Unsupported Rate | Safety | Notes |
|---|---|---|---|---|---|---|---|---|---|
| run-001 | 2026-07-17 | FAIL | 1.0 | 4.60 | 3.20 | 0.90 | 0.00 | 5.00 | First run; retrieval/faithfulness/citation/safety all pass on first try; relevancy (answer correctness) misses threshold — model under-specifies exact figures (dollar amounts, dates, counts) |
| run-002 | 2026-07-17 | FAIL | 1.0 | 4.61 | 3.00 | 0.90 | 0.00 | 5.00 | Corpus expanded 5→9 docs (~1.6k→~3.1k words) + eval set 20→36 rows to test whether denser numeric phrasing fixes relevancy; it didn't — correctness ticked down (3.2→3.0), not up. Next: isolate old-vs-new questions per-row, or revisit generation/judging setup rather than further corpus density |
