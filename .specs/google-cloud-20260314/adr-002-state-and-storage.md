# ADR-002: Persistent State Backend

**Status:** Proposed
**Date:** 2026-03-14

## Context

All state is currently local files:

| Data | Current Location | Access Pattern |
|------|-----------------|----------------|
| Archive state (grid cells, variant list) | `oe_state/map_elites_state.json` etc. | Read/write every iteration via `save_state()`/`load_state()` |
| Agent code variants | Embedded in archive JSON (`AgentVariant.code` field) | Written on mutation, read on parent selection |
| Experiment results log | `oe_results.tsv`, `results.tsv` | Append-only, one row per iteration |
| Evaluation traces | LangSmith (already cloud) | Write during eval, read via LangSmith UI |

This works for single-machine execution. It breaks with parallel workers because:
- Multiple workers writing to the same JSON file causes data loss
- TSV append from concurrent processes can interleave lines
- No shared access to archive state across machines

## Decision

Use **three GCP storage services**, each matched to its data's access pattern:

### 1. Cloud Storage (GCS) — Agent code variants

Store each `AgentVariant.code` blob as a GCS object: `gs://{bucket}/variants/{iteration}-{hash}.py`

- Code blobs are write-once, read-many (immutable after creation)
- GCS is ideal for blob storage with no query needs
- The `AgentVariant` dataclass stores a `gcs_uri` reference instead of inline `code` string
- Keeps archive JSON small (metadata only, no multi-KB code strings)

### 2. Firestore — Archive state and variant metadata

The `GridArchive` and `UnstructuredArchive` become Firestore-backed:

- **GridArchive**: Each cell is a Firestore document keyed by `(dim0_bin, dim1_bin)`. Fields: `fitness`, `iteration`, `description`, `descriptors`, `gcs_uri`. The `add()` method uses a Firestore transaction to compare-and-swap (only update if new fitness > existing).
- **UnstructuredArchive**: A single collection where each variant is a document. The `add()` method writes a new document; the `max_size` eviction runs as a periodic cleanup or transactional trim.
- **Algorithm metadata**: `iteration` counter, algorithm-specific config stored as a singleton document per search run.

Firestore provides:
- Transactions for safe concurrent `update_archive` from parallel workers
- Sub-10ms reads for `select_parent` (sampling from archive)
- Automatic indexing for queries like "top-k by fitness"

### 3. BigQuery — Experiment results log

Replace `oe_results.tsv` and `results.tsv` with a BigQuery table:

```
Table: experiments.results
Schema:
  run_id        STRING    -- unique search run identifier
  iteration     INT64
  overall_score FLOAT64
  correctness   FLOAT64
  helpfulness   FLOAT64
  tool_usage    FLOAT64
  status        STRING    -- "accepted" / "rejected"
  description   STRING
  descriptors   JSON
  variant_uri   STRING    -- GCS path to code
  timestamp     TIMESTAMP
  algorithm     STRING    -- "map-elites", "adas", etc.
  worker_id     STRING    -- which worker produced this
```

- Append-only writes from workers (BigQuery streaming insert)
- Rich analytical queries: "which behavioral niches have the highest mean fitness?", "what's the acceptance rate over time?"
- Replaces the `_log_result` method in `base.py:410`

## Alternatives Considered

### Cloud SQL (PostgreSQL)

Pros: Full relational model, ACID transactions, familiar.
Cons: Always-on instance cost (~$50/month minimum for a small instance), requires connection management, overkill for the data volume (hundreds to low thousands of variants).

**Rejected because**: Firestore's serverless model matches the bursty usage pattern. No reason to pay for an always-on database when experiments run intermittently.

### GCS for everything (JSON files in buckets)

Pros: Simplest possible approach, direct migration from local files.
Cons: No transactional updates — two workers writing the same archive JSON simultaneously causes data loss. No query capability for analytics. Would need external locking (e.g., GCS object generation-match), which is fragile.

**Rejected because**: The archive requires transactional compare-and-swap semantics that GCS alone cannot provide.

### Firestore for everything (including code blobs)

Pros: Single storage system.
Cons: Firestore document size limit is 1 MiB. Agent code files are small today (~3-5 KB) but could grow if agents become more complex. More importantly, Firestore charges per document read/write — storing large blobs there is cost-inefficient vs GCS.

**Rejected because**: GCS is cheaper and more appropriate for immutable blobs. Firestore for metadata + GCS for blobs is the standard pattern.

## Consequences

### Positive
- Parallel workers can safely read/write the archive concurrently
- Full experiment history is queryable (BigQuery) rather than locked in local TSV files
- State survives machine failure — no lost experiments
- Code variants are durably stored and retrievable (GCS)

### Negative
- `AgentVariant` dataclass gains a `gcs_uri` field; code is lazy-loaded instead of inline
- `GridArchive.add()` and `UnstructuredArchive.add()` become async-capable (Firestore calls)
- `save_state()`/`load_state()` in each algorithm class are replaced by Firestore operations
- Local development needs a Firestore emulator or a fallback to the current JSON-file approach
- GCP costs: Firestore free tier covers ~50K reads/day, BigQuery first 1TB/month is free, GCS costs are negligible for small text files

### Impact on Existing Code
- `archive.py:GridArchive.add()` — gains a Firestore transaction wrapper for compare-and-swap
- `archive.py:GridArchive.to_dict()`/`from_dict()` — replaced by Firestore serialization (or kept as a local-mode fallback)
- `archive.py:AgentVariant` — `code` field becomes optional; `gcs_uri` field added
- `base.py:_log_result()` — BigQuery streaming insert instead of file append
- `base.py:save_state()`/`load_state()` — Firestore read/write instead of JSON file I/O
- `map_elites.py`, `adas.py`, `novelty_search.py`, `go_explore.py` — `save_state`/`load_state` overrides updated
