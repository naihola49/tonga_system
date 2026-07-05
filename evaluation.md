← [Overview](README.md)

# Evaluation

Retrieval and RAG quality are measured on a hand-curated eval set with deterministic chunk-ID metrics and RAGAS LLM-as-judge scores.

**Full methodology and latest outcomes:** [ragas_tests/retrieval_evaluation.md](ragas_tests/retrieval_evaluation.md)

**Dataset (60 samples):** [ragas_tests/eval_dataset.json](ragas_tests/eval_dataset.json)

---

## Current snapshot — `legal_boundary_v4`, k=5

| Metric | v3 (60 samples) | v4 (60 samples) |
|--------|--------------:|----------------:|
| hit@k | 93.3% | **95.0%** |
| hit@1 | 75.0% | 75.0% |
| Chunk-ID recall | 93.3% | **95.0%** |
| RAGAS context recall | 87.5% | **89.2%** |
| RAGAS faithfulness | 52.3% | **60.7%** |

Historical baseline (30 samples, vector-only, pre-fusion v2): hit@k **73.3%** → hybrid fusion v3 **93.3%** (+20 pp), validating the three-leg retrieval design.

Level 4 policies remain the main retrieval challenge (80% hit@k); instructions and guidelines score 100% on this set.
