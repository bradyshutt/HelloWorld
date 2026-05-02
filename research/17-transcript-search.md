# Searchable Transcripts & Semantic Search

## Summary

For a voice journal, the strongest 2025/2026 approach is **hybrid retrieval**: pair a lexical/full-text index (BM25-style) for exact phrases, names, and dates with an embedding index for "find me when I felt anxious about deadlines"-style semantic queries. Since journals are personal and small-to-medium scale, an **embedded SQLite stack — FTS5 + sqlite-vec, fused with Reciprocal Rank Fusion (RRF)** — gives you offline-friendly, dependency-light search with near best-in-class quality. Word-level timestamps from Whisper (or `whisper-timestamped` / `stable-ts`) should be stored alongside each indexed segment so any hit can deep-link back to the exact second of audio playback.

## Full-Text Options

- **SQLite FTS5** — Built into SQLite, creates an inverted index in a virtual table, supports BM25 ranking, phrase/prefix/boolean queries, and `NEAR()` proximity. Ideal for embedded/offline-first apps and trivially co-located with the rest of the app DB. Has known caveats around tokenizer flexibility and write contention; Turso is building a Tantivy-based replacement that is worth tracking ([Beyond FTS5 — Turso](https://turso.tech/blog/beyond-fts5), [SQLite FTS5 in Practice](https://thelinuxcode.com/sqlite-full-text-search-fts5-in-practice-fast-search-ranking-and-real-world-patterns/)).
- **PostgreSQL `tsvector` / `tsquery`** — Server-side option with stemming, stopwords, and GIN indexes; great if the journal is multi-user/cloud and you already run Postgres. Combine with `pgvector` for hybrid in one DB ([Different ways to Search Text in PostgreSQL — Aiven](https://aiven.io/blog/different-ways-to-search-text-in-postgresql), [Hybrid Search in PostgreSQL — ParadeDB](https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual)).
- **Meilisearch** — LMDB-backed, disk-friendly, fastest DX, built-in admin UI, but BYO embeddings on OSS and primarily single-node ([Meilisearch vs Typesense](https://www.meilisearch.com/blog/meilisearch-vs-typesense)).
- **Typesense** — RAM-resident, sub-50 ms latency, built-in clustering (Raft), and auto-embedding via internal/external models — convenient if you want vector + lexical from one server without writing code ([Typesense alternatives 2026](https://www.meilisearch.com/blog/typesense-alternatives)).

For a single-user voice journal, FTS5 wins on simplicity; Typesense/Meilisearch only become attractive if you go cloud/multi-user.

## Semantic / Embedding Options

MTEB leaderboards in 2025/2026 put paid APIs (Cohere embed-v4 ~65, OpenAI `text-embedding-3-large` ~64.6, Voyage-3) at the top, with **BGE-M3** (~63, 100+ languages, 8192-token context) the strongest open-source choice and **E5-Base-v2 / BGE-Base-v1.5** offering ~84% accuracy at ~80 ms latency ([Embedding Models Comparison 2026 — Reintech](https://reintech.io/blog/embedding-models-comparison-2026-openai-cohere-voyage-bge), [Best Open-Source Embedding Models — BentoML](https://www.bentoml.com/blog/a-guide-to-open-source-embedding-models)).

For a voice journal:

- **Cloud / easiest**: OpenAI `text-embedding-3-small` (cheap, 62% MTEB, Matryoshka truncation lets you store 256–1536 dims) or Voyage-3.5 (RAG-tuned).
- **Local / private**: **BGE-small-en-v1.5** or **all-MiniLM-L6-v2** for speed (~15 ms / 1K tokens) when journals must never leave the device; MiniLM trades ~10–15 points of recall for tiny size and CPU-friendliness ([Top Embedding Models 2025 — Artsmart](https://artsmart.ai/blog/top-embedding-models-in-2025/)).
- **Storage**: `sqlite-vec` (successor to `sqlite-vss`) is pure C, dependency-free, supports KNN + multiple distance metrics, runs in WASM/mobile, and lives in the same DB file as FTS5 — perfect for a journal app ([sqlite-vec on GitHub](https://github.com/asg017/sqlite-vec), [Embedded Intelligence with sqlite-vec](https://dev.to/aairom/embedded-intelligence-how-sqlite-vec-delivers-fast-local-vector-search-for-ai-3dpb)).

Chunk transcripts into 1–3 sentence segments (or ~30–60 s of audio) so each embedding maps cleanly to a timestamp range.

## Hybrid Retrieval

Lexical and vector scores are not directly comparable (BM25 is unbounded, cosine is [0, 2]). The 2025 consensus is **Reciprocal Rank Fusion** because it is score-agnostic, needs no tuning, and resists outliers — `score = Σ 1 / (k + rank)` with k≈60 ([Hybrid Retrieval with RRF — Chauzov](https://avchauzov.github.io/blog/2025/hybrid-retrieval-rrf-rank-fusion/), [Hybrid full-text + vector search with SQLite — Alex Garcia](https://alexgarcia.xyz/blog/2024/sqlite-vec-hybrid-search/index.html), [Hybrid Search Scoring — Azure AI Search](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)).

A solid pipeline:

1. **Retrieve** top ~50 from FTS5 (BM25) + top ~50 from sqlite-vec (cosine).
2. **Fuse** with RRF down to ~20 candidates.
3. **(Optional) Rerank** with a small cross-encoder (e.g., `bge-reranker-base`) for the final 5–10.

Hybrid especially helps journals because users search both fuzzy moods ("when I was stressed") and exact tokens ("Dr. Patel", "March 14").

## Indexing Audio Timestamps

Use Whisper variants that emit **word-level timestamps** (`whisper-timestamped`, `stable-ts`, Groq's word-level API, or `easytranscriber`) ([whisper-timestamped](https://github.com/linto-ai/whisper-timestamped), [stable-ts](https://github.com/jianfch/stable-ts), [Groq word-level timestamping](https://groq.com/blog/build-fast-with-word-level-timestamping)). For each chunk, persist `{entry_id, audio_uri, start_ms, end_ms, text, embedding}`. FTS5 stores `text`, sqlite-vec stores the vector keyed by the same row id. Search results then carry `start_ms`, letting the UI deep-link straight into playback with a synced highlight — the same pattern FrameQuery and modern podcast tools use ([Transcription with Timecode — WhisperBot](https://whisperbot.ai/blog/transcription-with-timecode), [FrameQuery transcript search](https://www.framequery.com/features/transcription-search)).

## Recommendations

- **Default stack**: SQLite + FTS5 + `sqlite-vec`, single `.db` file per user, RRF fusion. Maximum portability, offline-first, easy backups.
- **Embeddings**: start with `text-embedding-3-small` (768 dims via Matryoshka) for cloud users; ship BGE-small or MiniLM-L6-v2 via ONNX/Transformers.js for local-only mode.
- **Chunking**: 1–3 sentence chunks tied to Whisper word timestamps; store `start_ms`/`end_ms` for jump-to-quote.
- **Upgrade paths**: swap to Postgres + pgvector + ParadeDB if multi-user/cloud, or to Typesense if you want managed auto-embeddings and clustering without writing fusion code yourself.
- **Future**: keep an eye on Turso's Tantivy-based FTS replacement and on cross-encoder rerankers small enough to run on-device.
