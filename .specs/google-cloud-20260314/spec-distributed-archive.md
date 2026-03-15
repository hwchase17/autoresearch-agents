# Spec: Distributed Archive Service

**Phase:** 2 (requires Phase 1 complete; prerequisite for Phase 3)
**Related ADRs:** [ADR-002](adr-002-state-and-storage.md), [ADR-004](adr-004-migration-strategy.md)

## Overview

Cloud-backed implementations of `GridArchive` and `UnstructuredArchive` from `algorithms/archive.py` that support concurrent read/write from multiple workers. Uses Firestore for archive metadata, GCS for agent code blobs, and BigQuery for experiment logging.

## Current Interfaces to Preserve

### AgentVariant (`archive.py:17-31`)

```python
@dataclass
class AgentVariant:
    code: str                           # full agent.py source
    scores: dict                        # {overall_score, avg_correctness, ...}
    descriptors: dict                   # {tool_usage_score, code_lines, ...}
    iteration: int = 0
    parent_iteration: int | None = None
    description: str = ""
    commit_hash: str = ""

    @property
    def fitness(self) -> float:
        return self.scores.get("overall_score", 0.0)
```

### GridArchive (`archive.py:39-128`)

```python
class GridArchive:
    def __init__(self, dims, ranges, resolutions): ...
    def add(self, variant: AgentVariant) -> bool: ...      # compare-and-swap on cell
    def sample(self) -> AgentVariant | None: ...            # random occupied cell
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data) -> GridArchive: ...
    @property
    def best(self) -> AgentVariant | None: ...
    @property
    def size(self) -> int: ...
    @property
    def coverage(self) -> float: ...
    def summary(self) -> dict: ...
```

### UnstructuredArchive (`archive.py:136-198`)

```python
class UnstructuredArchive:
    def __init__(self, max_size=500): ...
    def add(self, variant: AgentVariant) -> None: ...       # append + evict
    def sample(self, n=1) -> list[AgentVariant]: ...
    def sample_weighted_by_novelty(self, scores) -> AgentVariant | None: ...
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data) -> UnstructuredArchive: ...
    @property
    def best(self) -> AgentVariant | None: ...
    @property
    def size(self) -> int: ...
    def summary(self) -> dict: ...
```

## Firestore Data Model

### Collection: `archives/{run_id}`

Singleton document storing archive configuration:

```
{
    "algorithm": "map-elites",
    "dims": ["tool_usage_score", "correctness"],
    "ranges": [[0.0, 1.0], [0.0, 1.0]],
    "resolutions": [5, 5],
    "iteration": 42,
    "created_at": timestamp,
    "updated_at": timestamp
}
```

### Collection: `archives/{run_id}/cells` (GridArchive)

One document per occupied grid cell:

```
Document ID: "2_3"  (bin indices joined by underscore)
{
    "fitness": 0.883,
    "iteration": 17,
    "parent_iteration": 5,
    "description": "Added web search tool",
    "scores": {"overall_score": 0.883, "avg_correctness": 0.90, ...},
    "descriptors": {"tool_usage_score": 0.8, "correctness": 0.9, ...},
    "code_uri": "gs://autoresearch-variants/run-42/iter-17.py",
    "updated_at": timestamp
}
```

### Collection: `archives/{run_id}/variants` (UnstructuredArchive)

One document per variant:

```
Document ID: auto-generated
{
    "fitness": 0.75,
    "iteration": 8,
    "parent_iteration": 3,
    "description": "Switched to chain-of-thought prompting",
    "scores": {...},
    "descriptors": {...},
    "code_uri": "gs://autoresearch-variants/run-42/iter-8.py",
    "created_at": timestamp
}
```

## Cloud-Backed Archive Implementations

### FirestoreGridArchive

```python
class FirestoreGridArchive:
    """GridArchive backed by Firestore with transactional compare-and-swap."""

    def __init__(self, run_id: str, dims, ranges, resolutions, db=None):
        self.run_id = run_id
        self.dims = dims
        self.ranges = ranges
        self.resolutions = resolutions
        self.db = db or firestore.Client()
        self.collection = self.db.collection("archives").document(run_id).collection("cells")

    def add(self, variant: AgentVariant) -> bool:
        """Transactional compare-and-swap: only update if new fitness > existing."""
        idx = self._to_index(variant.descriptors)
        doc_id = "_".join(str(i) for i in idx)
        doc_ref = self.collection.document(doc_id)

        @firestore.transactional
        def update_in_transaction(transaction):
            snapshot = doc_ref.get(transaction=transaction)
            if snapshot.exists:
                existing_fitness = snapshot.get("fitness")
                if variant.fitness <= existing_fitness:
                    return False
            # Upload code to GCS, write metadata to Firestore
            code_uri = self._upload_code(variant)
            transaction.set(doc_ref, {
                "fitness": variant.fitness,
                "iteration": variant.iteration,
                "scores": variant.scores,
                "descriptors": variant.descriptors,
                "description": variant.description,
                "code_uri": code_uri,
                "updated_at": firestore.SERVER_TIMESTAMP,
            })
            return True

        transaction = self.db.transaction()
        return update_in_transaction(transaction)

    def sample(self) -> AgentVariant | None:
        """Random sample from occupied cells."""
        docs = list(self.collection.stream())
        if not docs:
            return None
        doc = random.choice(docs)
        return self._doc_to_variant(doc)
```

The critical property: `add()` uses a Firestore transaction so two workers writing to the same cell will serialize correctly. The higher-fitness variant always wins.

### FirestoreUnstructuredArchive

```python
class FirestoreUnstructuredArchive:
    """UnstructuredArchive backed by Firestore."""

    def add(self, variant: AgentVariant) -> None:
        """Append variant. Evict lowest-fitness if over max_size."""
        code_uri = self._upload_code(variant)
        self.collection.add({
            "fitness": variant.fitness,
            "iteration": variant.iteration,
            "scores": variant.scores,
            "descriptors": variant.descriptors,
            "description": variant.description,
            "code_uri": code_uri,
            "created_at": firestore.SERVER_TIMESTAMP,
        })
        # Eviction: query count, if over max_size, delete lowest-fitness
        self._maybe_evict()

    def sample(self, n=1) -> list[AgentVariant]:
        """Random sample from archive."""
        # Firestore doesn't have native random sampling.
        # Strategy: fetch all doc IDs, random.sample, fetch full docs.
        all_ids = [doc.id for doc in self.collection.select([]).stream()]
        chosen = random.sample(all_ids, min(n, len(all_ids)))
        return [self._doc_to_variant(self.collection.document(id).get()) for id in chosen]
```

## GCS Code Storage

Each `AgentVariant.code` blob is stored as a GCS object:

```
gs://autoresearch-variants/{run_id}/iter-{iteration}-{short_hash}.py
```

- Write-once, read-many (immutable after creation)
- Short hash is first 8 chars of SHA-256 of the code content (deduplication key)
- Code is loaded lazily: `AgentVariant.code` is populated on-demand from `code_uri`

## BigQuery Logging

Replace `base.py:_log_result()` (line 410) with BigQuery streaming insert:

```python
def _log_result(self, variant: AgentVariant, accepted: bool) -> None:
    if self.bq_client:
        row = {
            "run_id": self.run_id,
            "iteration": variant.iteration,
            "overall_score": variant.scores.get("overall_score", 0),
            "correctness": variant.scores.get("avg_correctness", 0),
            "helpfulness": variant.scores.get("avg_helpfulness", 0),
            "tool_usage": variant.scores.get("avg_tool_usage", 0),
            "status": "accepted" if accepted else "rejected",
            "description": variant.description,
            "descriptors": json.dumps(variant.descriptors),
            "variant_uri": variant.gcs_uri,
            "timestamp": datetime.utcnow().isoformat(),
            "algorithm": self.__class__.__name__,
            "worker_id": self.worker_id,
        }
        self.bq_client.insert_rows_json(self.bq_table, [row])
    else:
        # existing TSV append logic
        ...
```

## Local Fallback

Both `FirestoreGridArchive` and `FirestoreUnstructuredArchive` implement the same interface as their local counterparts. A factory function selects the backend:

```python
def create_grid_archive(dims, ranges, resolutions, *, cloud=False, run_id=None):
    if cloud:
        return FirestoreGridArchive(run_id, dims, ranges, resolutions)
    else:
        return GridArchive(dims, ranges, resolutions)
```

Algorithm classes (`map_elites.py`, etc.) call the factory instead of constructing archives directly.

## Files to Create/Modify

| File | Action | Description |
|------|--------|-------------|
| `algorithms/cloud_archive.py` | Create | `FirestoreGridArchive`, `FirestoreUnstructuredArchive` |
| `algorithms/cloud_storage.py` | Create | GCS upload/download helpers, BigQuery logging |
| `algorithms/archive.py` | Modify | Add `gcs_uri` field to `AgentVariant`, add factory functions |
| `algorithms/base.py` | Modify | `_log_result()` gains BigQuery path; `save_state`/`load_state` become no-ops in cloud mode (Firestore is always current) |
| `algorithms/map_elites.py` | Modify | Use archive factory instead of direct `GridArchive()` construction |
| `algorithms/adas.py` | Modify | Same — use archive factory |
| `algorithms/novelty_search.py` | Modify | Same |
| `algorithms/go_explore.py` | Modify | Same |
| `run_search.py` | Modify | Add `--storage local|cloud`, `--run-id`, `--gcp-project` flags |
| `requirements.txt` | Modify | Add `google-cloud-firestore`, `google-cloud-storage`, `google-cloud-bigquery` |

## Verification

1. Run MAP-Elites with `--storage cloud --run-id test-1 --max-iterations 5`
2. Confirm Firestore contains `archives/test-1/cells` documents
3. Confirm GCS contains code blobs under `gs://autoresearch-variants/test-1/`
4. Confirm BigQuery table has 5+ rows for `run_id = "test-1"`
5. Stop the process, restart with same `--run-id`, confirm it resumes from correct iteration
6. Run the same test with `--storage local` and confirm identical scores (same agent mutations, same eval results)
