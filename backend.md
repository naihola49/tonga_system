← [Overview](README.md)

# Backend

The backend is a **FastAPI** service on **GCP Cloud Run**, Python 3.13.

## Parent–leaf indexing

Tonga Public Service policy material is hierarchically structured. Acts contain parts and sections, regulations contain sub-regulations, instructions nest under numbered units, and so on. Fixed-size text windows ignore that structure and produce chunks that span arbitrary boundaries, which breaks citation fidelity and confuses retrieval.

I indexed the corpus as a parent/leaf model:

| Tier | What it holds | Role in search |
|------|---------------|----------------|
| **Leaves** | Smallest legal units — subsections, sub-regulations, instruction clauses, definition entries | **Embedded and searched** (vectors + BM25). These are the precise citation anchors. |
| **Parents** | Section headers, part containers, instruction groups — stored with full text but **not embedded** | **Tracked in a hierarchy sidecar** (540 nodes) linked from each leaf via `parent_id` |

Every leaf carries metadata pointing to its parent and a human-readable hierarchy path (document → part → section → clause). Retrieval always ranks leaves first; when a leaf hits, the system can expand upward to attach parent context for the LLM or UI without diluting the vector index with duplicate or oversized parent blobs.

This strategy enables richer grounding: parent expansion gives the generator surrounding context (e.g. the full section around a single sub-clause) without merging siblings that should rank independently. The parent–leaf split is what makes retrieval-unit deduplication meaningful: siblings under the same section share a parent but remain distinct searchable leaves.

The hierarchy sidecar is a separate JSON artifact shipped with each index version alongside the vector store and BM25 index. It holds the 540 parent nodes — their text, child links, and structural metadata — while the searchable index stays leaf-only. At runtime the backend loads the sidecar into memory; when a retrieved leaf matches, it looks up that leaf’s parent chain and optionally attaches parent text to the result before generation or display. Parents never enter the embedding index, so search stays precise on small legal units, but the sidecar still makes full section context available on demand without re-fetching or re-chunking PDFs.

## Hybrid fusion retrieval

Instead of relying on embedding similarity alone, the backend runs three parallel retrieval legs and merges them with a custom weighted reciprocal rank fusion (RRF) layer I built for the legal corpus.


### Leg 1 — Dense vector search (cosine similarity)

- Chunks are embedded with OpenAI `text-embedding-3-small`.
- Stored in **ChromaDB** with cosine distance.
- Strong for paraphrase and conceptual queries (“accountability in appointments”).

### Leg 2 — BM25 keyword search

- Separate in-memory **BM25** index over the same 1,622 leaves.
- Index text is enriched with document title, hierarchy path, and definition terms — not raw body text alone.
- Strong for exact legal tokens (section numbers, policy IDs, named instruments).

### Leg 3 — Metadata search

- Rule-based leg for **structured legal signals** in the query:
  - Citation patterns (Section 5(b), Regulation 6, Roman numerals, decimal refs)
  - Definition intent (“define a Ministry”, “definition of …”)
- Scores chunks by matching query tokens against chunk metadata (subsection refs, definition terms, hierarchy path) rather than full-text similarity alone.

### Fusion layer

The three ranked lists are combined with **weighted RRF**:

- Default weight **1.0** for vector and BM25 legs.
- **2.5× weight** for the metadata leg when the query contains citation or definition signals; otherwise metadata stays at 1.0 so vague queries are not over-steered.

After fusion, **retrieval-unit deduplication** runs: distinct legal subsections (e.g. `4(1)` vs `4(d)`) are kept as separate hits instead of collapsing siblings under one parent section.

Finally, **parent expansion** can attach parent-section context from a hierarchy sidecar (540 parent nodes) so the LLM or UI sees broader statutory context without re-ranking away the correct leaf.

### Why fusion instead of a single retriever?

| Query type | Best leg |
|------------|----------|
| “Section 14(2) Minister consultation” | Metadata + BM25 |
| “How does the Act define a Ministry?” | Metadata (definition) + vectors |
| “Accountability before appointments” | Vectors + BM25 |
| Vague procedural wording | Vectors (metadata weight unchanged) |

No single scorer wins across all six policy levels; fusion recovers recall without hand-tuned rules per query.

## Generation layer

- OpenAI chat model.
- Retrieved **leaf text** is preferred over parent-expanded blobs when packing context — parent-expanded content can exceed judge token limits in evaluation.

## Startup and index hydration

Cloud Run containers are **stateless**. On boot:

1. Compare configured `INDEX_VERSION` to the on-disk manifest.
2. If mismatch (or empty disk), download the versioned **vector snapshot tarball** from GCS.
3. Verify **SHA-256** checksum before extract.
4. Load ChromaDB data, BM25 JSON index, and hierarchy sidecar into memory.

PDFs mount from a separate GCS bucket (read-only volume) for source linking — not embedded in the API image.

## Data layer

### Source documents

Fifteen policy PDFs across six levels:

| Level | Domain | Approx. leaves |
|-------|--------|----------------|
| L1 | Public Service Act (+ amendments) | 261 |
| L2 | Code of Ethics | 46 |
| L3 | Regulations (disciplinary, grievance, etc.) | 199 |
| L4 | Policies (fraud, harassment, etc.) | 260 |
| L5 | Policy Instructions | 752 |
| L6 | Guidelines (social media, neutrality, etc.) | 104 |

**Total: 1,622 embeddable leaf chunks**, plus **540 parent nodes** in the hierarchy sidecar.

### Chunking philosophy

Current approach: legal-boundary chunking — splits at sections, subsections, regulations, instruction units, and guideline clauses. Each leaf carries rich metadata:

- Document level and name  
- Chunk type (subsection, subregulation, instruction unit, etc.)  
- Parent ID and hierarchy path  
- Section/subsection references  
- Definition terms where applicable  

### Index artifacts (per version)

Each released index version ships as a tarball containing:

| Artifact | Role |
|----------|------|
| ChromaDB store | Dense vectors + chunk metadata |
| BM25 index | Serialized keyword index with enriched text fields |
| Hierarchy sidecar | Parent/child graph for expansion |
| Index manifest | Version, chunk counts, embedding model, build timestamp |

Built offline (not at request time), uploaded to GCS, referenced by deploy configuration.
