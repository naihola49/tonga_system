← [System design overview](../README.md)

# Retrieval Evaluation

The eval dataset lives in this directory: [`eval_dataset.json`](eval_dataset.json).

---

## Purpose

Retrieval evaluation answers two questions before we trust generated answers:

1. **Did search return the right legal passage?** — measured with deterministic chunk-ID overlap (no LLM judge).
2. **Is the full RAG pipeline grounded and complete?** — measured with RAGAS metrics that compare retrieved context and generated answers against human-written references.

The corpus spans six policy levels (L1 Acts through L6 Guidelines). Retrieval must work across acts, regulations, policies, instructions, and guidelines—not only on easy citation-style queries.

---

## Evaluation dataset

| Property | Value |
|----------|-------|
| **Samples** | 60 (10 per level L1–L6) |
| **Schema** | Question, reference answer, one or more anchor chunk IDs |
| **Curation** | Hand-written; each anchor verified against the indexed corpus |
| **Metadata per sample** | Document level, topic, difficulty (`easy` / `medium` / `hard`) |
| **Index version** | `legal_boundary_v4` (1631 embedded leaf chunks) |

Samples cover definition queries (“How does the Act define …?”), citation-style questions (“What does Section 4B say …?”), and natural operational questions (“How are permanent employees appointed …?”) with no explicit section number.

---

## Methodology

### Pipeline

Each sample is run against a live backend local instance through two phases:

```
eval_dataset.json
       │
       ▼
  Phase 1 — /search (k=5)
       │   Hybrid retrieval: vector + BM25 + metadata legs,
       │   weighted reciprocal-rank fusion, retrieval-unit dedup
       ▼
  Deterministic scoring (chunk-ID metrics)
       │
       ▼
  Phase 2 — /rag (full generation)
       │   Same retrieval context fed to the policy LLM
       ▼
  RAGAS scoring (LLM-as-judge, gpt-4o-mini)
```

**Retrieval settings:** top **k = 5** chunks per query. Optional level filter matches the sample’s document level (L1–L6).

**Generation settings:** fast chat model with faithfulness-oriented system instructions (answer only from provided context; low temperature).

### Deterministic retrieval metrics

These require no OpenAI key and are fully reproducible given the same index and query.

| Metric | Definition |
|--------|------------|
| **hit@k** | `1` if any anchor chunk ID appears anywhere in the top-k retrieved set; else `0` |
| **hit@1** | `1` if the top-ranked chunk is an anchor; else `0` |
| **Chunk-ID recall** (`id_based_context_recall`) | Fraction of anchor IDs found in the retrieved set (all anchors must appear for `1.0` when multiple are listed) |

### RAGAS metrics

Scored after full `/rag` generation using RAGAS 0.4 collections metrics and **gpt-4o** as judge:

| Metric | What it measures |
|--------|------------------|
| **Context recall** | Whether the retrieved passages contain enough information to support the reference answer |
| **Faithfulness** | Whether the generated answer is supported by the retrieved context (no hallucination) |

RAGAS scores are useful for end-to-end quality but can diverge from chunk-ID recall when the correct chunk is retrieved but the model misuses it (observed on a small number of L4 samples).

### Reporting

Results are written per run to `ragas_metrics/results/run_<timestamp>/` as CSV and JSON, with a `latest_results.csv` symlink for quick access. Aggregate means and per-level breakdowns are printed at the end of each run.

---

## Index evolution (retrieval-focused)

| Index | Key retrieval change | 30-sample hit@k | 60-sample hit@k |
|-------|----------------------|-----------------|-----------------|
| **v2** (baseline) | Vector-only | 73.3% | — |
| **v3** | Three-leg hybrid fusion + BM25 sidecar | 93.3% | 93.3% |
| **v4** | L4 definition leaves, BM25 policy/section aliases, metadata-leg tuning | — | **95.0%** |

The jump from v2 → v3 validated hybrid fusion (+20 pp). v4 targeted Level 4 policy documents, where definition blocks and implicit “Policy N” references were under-performing.

---

## Outcomes — `legal_boundary_v4` (July 2026)

**Run:** `run_20260704T233111Z` · 60 samples · k=5 · 1631 documents indexed

### Aggregate

| Metric | Score |
|--------|------:|
| hit@k | **95.0%** |
| hit@1 | **75.0%** |
| Chunk-ID recall | **95.0%** |
| RAGAS context recall | **89.2%** |
| RAGAS faithfulness | **60.7%** |

Compared to the prior v3 run on the same 60-sample set: hit@k +1.7 pp, chunk-ID recall +1.7 pp, context recall +1.7 pp, **faithfulness +8.4 pp** (prompt + retrieval improvements).

### By document level

| Level | hit@k | hit@1 | Chunk-ID recall | Context recall | Faithfulness |
|-------|------:|------:|----------------:|---------------:|-------------:|
| L1 Acts | 100% | 70% | 100% | 85.0% | 61.1% |
| L2 Code of Ethics | 100% | 70% | 100% | 95.0% | 60.5% |
| L3 Regulations | 90% | 70% | 90% | 90.0% | 50.7% |
| L4 Policies | 80% | 40% | 80% | 70.0% | 52.7% |
| L5 Instructions | 100% | 100% | 100% | 100% | 65.4% |
| L6 Guidelines | 100% | 100% | 100% | 95.0% | 73.6% |

### What improved in v4

- **L4 definition queries** — Interpretation sections (e.g. Fraud Policy “bribe”) are indexed as per-term definition leaves; previously missed definition samples now hit@1.
- **Faithfulness** — System prompt and temperature tuning reduced unsupported elaboration; largest gain at aggregate level (+8.4 pp).
- **Overall retrieval** — 57/60 samples retrieve at least one anchor in top-5 (up from 56/60 on v3).

---

## Interpretation guidelines

- **hit@k ≥ 90%** is the primary retrieval success criterion: the user’s answer can be grounded if any anchor appears in context.
- **hit@1** is stricter and sensitive to subsection siblings sharing a parent section; useful for diagnosing ranking, not a hard deploy gate.
- **Faithfulness ~60%** reflects RAGAS strictness on partial paraphrase and list-style answers; it improved materially with prompt changes and should be tracked alongside retrieval, not in isolation.
- Eval runs are **index-version-specific**; always record `index_version` from the dataset header and backend health check when comparing runs.
