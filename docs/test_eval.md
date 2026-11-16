# Test-split evaluation (Day 7 Agentic RAG)

Test split: 11 questions (8 answerable incl. 1 multi-hop t08, 3 unanswerable t09-t11). Test was run only after the config was frozen and nothing was tuned on it.

## Freeze evidence
- `configs/final.json`: `frozen: true`, `frozen_on: 2026-11-16`, derived from dev config `rerank_fix`. Only commit touching it: `52d74f3` "Record agent dev experiments and freeze final config"; file mtime 2026-11-16 10:20 +07, unmodified in the working tree. Test runs started 2026-11-16 23:43:16 +07 (HEAD `bf9113c`, same tree as the pre-rewrite `a08ca85`), i.e. after the freeze.
- Model `gemini-3.5-flash-lite` (Gemini free tier, thinking_budget 1024, temperature 0, pace_s 6.5), prompt v1, system prompt sha256 `d5ef139d76b5bf15096ea3ac10888aab48a7a5a13813ed06124d8695013db948`. Embedding BAAI/bge-small-en-v1.5, reranker cross-encoder/ms-marco-MiniLM-L-6-v2, collection `soc3e_B`.
- final: hybrid_rerank (pool/fuse depth 30, RRF k 60, demote_backmatter), top_k 5, max 2 tool calls, context budget 8000 chars. base: hybrid depth 20, no rerank, top_k 5, max 3 tool calls, same model/prompt.
- No gold is passed to the agent (`run_agent.py` reads only `benchmark/questions.json`); gold is read only by `eval_retrieval.py`, `eval_agent.py`, `review_dump.py`.

## Commands (repo root, `.venv/Scripts/python.exe`, `PYTHONPATH=.`)
1. Sanity (no API, dev, in-memory, nothing written): re-ran `imp6_hybrid_rerank_pool30_demote_k5`, `hybrid_k5`, `dense_k5` on dev; identical to saved dev summaries (imp6 Hit@1 1.000 / Hit@3 1.000).
2. `scripts/eval_retrieval.py --split test --allow-test --configs dense_k5,bm25_k5,hybrid_k5,hybrid_rerank_k5,imp6_hybrid_rerank_pool30_demote_k5`
3. `scripts/run_agent.py --split test --config final --allow-test`, then `--config base --allow-test` (started 2026-11-16 23:43:16 +07, about 6 min in total; no resume used).
4. Manual grading with `scripts/review_dump.py test final,base t01,...,t11`; labels in `results/agent/test/<config>/reviews.json`.
5. `scripts/eval_agent.py --split test --configs final,base --allow-test` produced `summary.json` per config and `results/agent/test/comparison.md`.

## Retrieval (test, original query, collection B, k=5)
| config | Hit@1 | Evidence Hit@3 | Hit@5 | MRR | latency med / p95 ms | failed |
|---|---|---|---|---|---|---|
| dense_k5 (Day 6 baseline) | 0.875 (7/8) | 0.875 (7/8) | 0.875 | 0.875 | 23 / 30 | t03 |
| bm25_k5 | 0.750 (6/8) | 0.875 (7/8) | 0.875 | 0.812 | 0 / 1 | t03 |
| hybrid_k5 | 0.875 (7/8) | 0.875 (7/8) | 0.875 | 0.875 | 23 / 26 | t03 |
| hybrid_rerank_k5 | 0.875 (7/8) | 0.875 (7/8) | 1.000 | 0.906 | 727 / 934 | none |
| **imp6 hybrid_rerank pool30 demote (chosen)** | 0.875 (7/8) | **1.000 (8/8)** | 1.000 | 0.917 | 944 / 1515 | none |

Dev to test for the chosen config: Hit@1 1.000 -> 0.875, Hit@3 1.000 -> 1.000. Retrieval latencies were measured while the first agent run was active, so they may be slightly inflated. Reranking costs about 1 s/query on CPU versus about 25 ms for hybrid.

## Agent (test, hand-graded)
| metric | base | final |
|---|---|---|
| provider errors | 0 | 0 |
| tool use (called retrieval / ran) | 11/11 | 11/11 |
| tool-call distribution (1/2/3 calls) | 5/3/3 | 7/4/0 |
| tool calls / query | 1.82 | 1.36 |
| context evidence rate (gold in final context) | 8/8 | 8/8 |
| grounded answer 0/1/2 (answerable) | 0/0/8 | 0/0/8 |
| mean /2 | 2.000 | 2.000 |
| citation valid (in context and supports claim) | 9/9 | 9/9 |
| answers with citation | 8/8 | 8/8 |
| abstention (unanswerable) | 3/3 | 3/3 |
| false abstention (answerable refused) | 0/8 | 0/8 |
| tokens / query (mean total; prompt) | 5697 (5193) | 4063 (3642) |
| LLM calls / query | 2.91 | 2.36 |
| mean final context chars | 5771 | 5292 |
| LLM+tool active time median / p95 s | 7.1 / 22.2 | 9.5 / 21.2 |
| wall time median s (incl. 6.5 s pacing) | 26.1 | 31.4 |
| cost | 0 USD (free tier); about 5.7k tokens/q | 0 USD (free tier); about 4.1k tokens/q |

### Context grade (BENCHMARK sec. 2; all 8 answerable, read from final_context)
| config | grade 0 | grade 1 | grade 2 |
|---|---|---|---|
| base | 0 | 0 | 8 |
| final | 0 | 0 | 8 |

Stored as extra fields `context_grade` / `context_reason` in each reviews.json (the aggregator ignores them; there was no existing tooling for this). Grade 2 means the gold chunk(s), with section/page source, are in the final context and suffice for the answer. For multi-hop t08 (evidence mode "all") both p0200-001 and p0201-000 were in context for both configs.

## Per-query (base and final cited the same chunks with equivalent answers on every query; tools shown as base/final)
| id | type | tools (base/final) | answer gist (citations) | labels | reason |
|---|---|---|---|---|---|
| t01 | direct | 1/1 | Erikson: eight stages, birth to death (p0133-004) | ans 2, cite ok, ctx 2 | chunk states it verbatim |
| t02 | direct | 2/1 | Durkheim: social (religious) causes, not psychological (p0134-002) | 2, ok, ctx 2 | chunk states it |
| t03 | direct | 2/1 | Elyse Schein and Paula Bernstein, reunited 2003 (p0137-001, -002) | 2, both ok, ctx 2 | 001 = twin study and 1968 separation, 002 = names and 2003 reunion; both in context |
| t04 | direct | 1/1 | peer group = similar age/status, shared interests (p0141-000) | 2, ok, ctx 2 | verbatim definition |
| t05 | direct | 3/2 | McDonaldization: Ritzer 1993, fast-food model spreading into institutions (p0174-000); base also lists efficiency/predictability/calculability/control | 2, ok, ctx 2 | chunk supports all; question asks only who/what |
| t06 | direct | 1/1 | Sutherland: differential association theory (p0196-001) | 2, ok, ctx 2 | chunk states it |
| t07 | direct | 1/1 | jail temporary/small/local vs prison >1 year, large, state/federal (p0203-002) | 2, ok, ctx 2 | every part in chunk |
| t08 | multi-hop | 1/1 | NCVS higher than UCR because it picks up unreported crimes (p0201-000) | 2, ok, ctx 2 (both evidence chunks) | answer needs only 0201-000; the source's hedge "likely" is preserved |
| t09 | unanswerable | 3/2 | abstained: Stanford prison experiment | abstention ok | no citations, no claim; gold: never mentioned |
| t10 | unanswerable | 3/2 | abstained: death penalty states | ok | same |
| t11 | unanswerable | 2/2 | abstained: Egalia enrollment | ok | Egalia is in the corpus (p136) but no number is given |

## Errors
None: 0 provider errors, 0 vector-store errors, no 429s, no retries, no resume.

## Discussion
- Both configs reached the ceiling on answer quality (8/8 grade 2, 9/9 valid citations, 3/3 abstentions), so the test set does not separate them on correctness. The measurable difference is efficiency: final used 1.36 vs 1.82 tool calls/q and about 29% fewer tokens/q (4063 vs 5697), and never needed a third call.
- Reading the search events: base's extra calls were partly useful and partly wasteful. On t02 base's first query missed the gold chunk (0134-002 only appeared in the second query), whereas final's first query ranked it first. On t05 the gold chunk was already rank 1 in base's first call, and the extra "Ritzer" / "George Ritzer" calls were redundant. On t03 both configs' agent-written query ("reunited twins nature nurture") already retrieved 0137-002 (rank 4 for base, rank 3 for final); base then issued a second query anyway. The 3 abstentions cost base an extra search each on t09/t10.
- Retrieval: with the raw benchmark question, all unreranked methods miss t03 in top-5, while both reranked configs recover it and only the chosen config has Hit@3 8/8. That did not turn into a higher answer grade, because the agent rewrites the query and still found the evidence (context evidence 8/8 for both). Retrieval metrics on the raw question and agent context metrics therefore measure different things.
- Dev to test: dev Hit@1 1.000 for the chosen config was optimistic (dev was used to choose it); test is 0.875. The dev agent differences were also small.
- Abstention was clean on all 3 unanswerable items, including the near-topic t11, but N=3.
- Limitations: tiny N (8 + 3), so ceiling results have wide uncertainty and cannot rank the configs on accuracy; single grader (Claude-assisted reading of each cited chunk against gold, student spot-check still required; labels were not independently double-checked); questions are mostly single-chunk lookups, so multi-hop and robustness are barely tested (one multi-hop item, answerable from one chunk); wall latency includes free-tier pacing sleeps; retrieval timings overlapped with an agent run; cost is 0 USD on the free tier, so tokens/query is the only cost proxy.
