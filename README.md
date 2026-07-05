# Overview

Deployed for the **Tongan Government Public Service Commission** (PSC).

## The Problem:

There are 300+ officials across 16 ministries working within the Kingdom of Tonga. Any questions on interpretation or situational application of policy require officials to consult the PSC's three-person policy team. The turnaround time is consistently between 3-6 business days.

## The Result:

A 20+ hrs/week reduction in manual policy research at the Public Service Commission and real-time answers. The policy team based in Nuku'Alofa, Tongatapu, Tonga now has increased bandwidth to review their policies. The general public can now directly query and engage with their laws. 

I am grateful to have this project been entrusted to me by the Kingdom of Tonga. The work was meaningful as I got to contribute and help my father's homeland.


Local news report: https://talanoaotonga.to/ai-chatbot-launched-to-streamline-public-service-policy-guidance/

## Solution Overview

| Concern | Approach |
|---------|----------|
| **Domain** | Six-level legal/policy hierarchy (L1–L6), 15 source PDFs |
| **Retrieval** | Custom three-leg hybrid search with weighted rank fusion |
| **Generation** | OpenAI LLM with policy-specific system instructions |
| **Auth** | Firebase Authentication |
| **Production** | Backend on GCP Cloud Run; frontend on Firebase App Hosting |
| **Index versioning** | Snapshot-based deploy (`legal_boundary_v4`) with GCS tarball + SHA verification |

## System Flow

```
User (browser)
    │
    ▼
Frontend ── Firebase Auth ──► Backend API
                                │
                                ▼
                 Hybrid search (3 parallel legs)
                                │
         ┌──────────────────────┼──────────────────────┐
         ▼                      ▼                       ▼
   Dense vectors              BM25                Metadata leg
   (cosine sim)             (keyword)           (citations, defs)
         │                      │                       │
         └──────────────────────┴───────────────────────┘
                                │
                      Weighted rank fusion
                                │
               Dedup + parent expansion (optional)
                                │
                          Top-k chunks
                                │
                                ▼
                               LLM
                                │
         ┌──────────────────────┴──────────────────────┐
         ▼                                              ▼
   RAG answer (streamed)                       PDF source links
                                             (from chunk metadata)
```
