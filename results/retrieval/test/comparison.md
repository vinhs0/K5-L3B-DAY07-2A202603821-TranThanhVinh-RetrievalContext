# Retrieval comparison (test split)

Answerable n=8 (multi-hop 1), unanswerable n=3. Hit@k = any gold chunk in top-k; Recall@k = fraction of required evidence; FullRecall = all required evidence. Latency = per-query wall time on CPU, warm models (ms).

| config | method | k | coll | Hit@1 | Hit@3 | Hit@5 | Recall@k | FullRecall@k | multi-hop full | MRR | lat med ms | lat p95 ms | errors | failed |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| dense_k5 | dense | 5 | B | 0.875 | 0.875 | 0.875 | 0.875 | 0.875 | 1.0 | 0.875 | 23 | 30 | 0 | t03 |
| bm25_k5 | bm25 | 5 | B | 0.750 | 0.875 | 0.875 | 0.875 | 0.875 | 1.0 | 0.812 | 0 | 1 | 0 | t03 |
| hybrid_k5 | hybrid | 5 | B | 0.875 | 0.875 | 0.875 | 0.875 | 0.875 | 1.0 | 0.875 | 23 | 26 | 0 | t03 |
| hybrid_rerank_k5 | hybrid_rerank | 5 | B | 0.875 | 0.875 | 1.000 | 1.000 | 1.000 | 1.0 | 0.906 | 727 | 934 | 0 |  |
| imp6_hybrid_rerank_pool30_demote_k5 | hybrid_rerank | 5 | B | 0.875 | 1.000 | 1.000 | 1.000 | 1.000 | 1.0 | 0.917 | 944 | 1515 | 0 |  |

## Unanswerable top-1 score vs answerable top-1 score (abstention signal)

| config | unans top1 mean | unans top1 max | ans top1 mean | ans top1 min |
|---|---|---|---|---|
| dense_k5 | 0.6111 | 0.6583 | 0.7789 | 0.6751 |
| bm25_k5 | 13.7762 | 22.9547 | 18.9690 | 8.1751 |
| hybrid_k5 | 0.0324 | 0.0328 | 0.0328 | 0.0325 |
| hybrid_rerank_k5 | -4.4588 | 2.5479 | 6.4743 | 2.5945 |
| imp6_hybrid_rerank_pool30_demote_k5 | -4.4588 | 2.5479 | 6.4743 | 2.5945 |

Scores are not comparable across methods (cosine / BM25 / RRF / cross-encoder logit).
