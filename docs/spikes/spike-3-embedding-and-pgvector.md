# Spike 3 — Embedding-model selection + pgvector HNSW recall/latency + model-swap

**Status:** Done — **SUCCESS**
**Date:** 2026-05-28
**Target epic:** Epic 7 — Memory Schemas + pgvector + HNSW Search
**Validates:** NFR2 (sub-50ms p95 memory search), the memory-layer embedding choice, `docs/architecture.md` §7
**Environment:** `pgvector/pgvector:pg16`, FastEmbed (ONNX, **no torch**), Python 3.11 (local Docker; models from HuggingFace, no API key)

## Finding that reshaped the spike

FastEmbed does **not** support `intfloat/multilingual-e5-small` or `google/embeddinggemma-300m` (the candidates the spec originally named). Using them would require `sentence-transformers` + `torch` (~2GB+), defeating the reason FastEmbed was chosen (keep `torch` out of the distroless image). The bake-off therefore pivoted, with user approval, to the FastEmbed-supported multilingual models.

## Result

Corpus: 200 docs (12 topics × {English rule, Thai rule} canonical + distractors), 26 labelled paraphrase queries.

| model | dim | size | retrieval p95 @200 | embed (CPU) | EN recall@10 | TH recall@10 | cross-lingual (TH→EN) |
|---|---|---|---|---|---|---|---|
| paraphrase-multilingual-MiniLM-L12-v2 | 384 | 0.22 GB | **0.46 ms** | 3.4 ms/doc | 92% | **100%** | 92% |
| paraphrase-multilingual-mpnet-base-v2 | 768 | 1.0 GB | 1.88 ms | 12.2 ms/doc | **100%** | **100%** | **100%** |

## Key findings

1. **NFR2 passes by ~100×.** p95 retrieval at 200 rows is 0.46/1.88 ms vs the 50ms target. pgvector + HNSW (`ef_search=40`, defaults) is over-provisioned for MVP scale; no tuning needed.
2. **Thai recall is 100% on both** — the user-zero Thai-SMB requirement is met with no torch.
3. **MiniLM-384** is tiny (0.22 GB) with 92% EN / 100% TH / 92% cross-lingual. **mpnet-768** is 1 GB with a perfect 100% across the board.
4. Both run on the FastEmbed ONNX runtime — **no `torch`**, keeping the distroless image small (feeds Spike 8).

## Recommended default

**`paraphrase-multilingual-MiniLM-L12-v2`, `VECTOR(384)`** for v0.4: smallest footprint (honors the 5-minute-install + zero-paid-key OSS promise), perfect Thai recall, sub-ms retrieval. **`paraphrase-multilingual-mpnet-base-v2` (768d)** documented as the higher-recall swap (100% all metrics) for operators who want maximum English/cross-lingual recall and accept ~1 GB.

## Model-swap + column-dim strategy

- `EMBEDDING_MODEL` env var selects the model. The install-time Alembic migration creates `memory_*.embedding VECTOR(N)` with N = the chosen model's dim (**fixed-per-deploy**).
- Switching models after install = a re-embed migration (rare, documented). No dynamic-dim scheme (avoided over-engineering).

## Gotchas (Epic 7 / Spike 8)

- **Pin `fastembed`.** It warned that MiniLM/mpnet pooling changed (mean vs CLS) across versions — pin the version in Epic 7 for reproducible embeddings.
- **Bake the model into the image.** HF downloads are unauthenticated + rate-limited; for the distroless cold-boot (Spike 8) bake MiniLM (0.22 GB) into the image rather than downloading on first boot — removes a network dependency from the 5-minute-install path.

## Follow-up — DONE (user-approved 2026-05-28)

**MiniLM-384 locked as the v0.4 default** (`VECTOR(384)`, fixed-per-deploy), mpnet-768 documented swap. Applied to FR21 (`docs/spec.md`), `docs/architecture.md` §7, and the `research_technical.md` embedding row. Licenses verified Apache 2.0 for both models (safe to bake into the OSS-distributed image).

## Decision

Memory layer validated. Default `MiniLM-384`; mpnet-768 documented swap; column dim fixed-per-deploy.
