# Spec: Distributed Search Algorithm Execution

**Phase:** 3 (requires Phase 1 + Phase 2)
**Related ADRs:** [ADR-001](adr-001-compute-platform.md), [ADR-003](adr-003-orchestration.md), [ADR-004](adr-004-migration-strategy.md)

## Overview

Decomposes the sequential `OpenEndedSearch.run()` loop (`algorithms/base.py:84-146`) into a distributed pipeline where multiple workers execute mutations and evaluations in parallel, coordinated via Pub/Sub, with a single Archive Updater ensuring consistency.

## Current Sequential Loop

```python
# algorithms/base.py:105-146
while True:
    self.iteration += 1
    parent = self.select_parent()                    # ~10ms (local dict lookup)
    new_code, description = self.mutate(parent)      # ~5-15s (LLM API call)
    scores, descriptors = self.evaluate(new_code)    # ~30s cloud / ~5min local
    variant = AgentVariant(code=new_code, scores=scores, ...)
    accepted = self.update_archive(variant)          # ~10ms (local dict update)
    self._log_result(variant, accepted=accepted)
    self.save_state()
```

**Total per iteration:** ~35-45 seconds (with Phase 1 cloud eval) or ~5-6 minutes (local eval).
**With N parallel workers:** ~35-45 seconds for N iterations (limited by the slowest worker, not N * latency).

## Distributed Pipeline Architecture

```
┌─────────────┐    mutation-requests    ┌────────────────┐
│  Dispatcher  │ ────────────────────→  │ Mutate Workers  │ (N instances)
│  (single)    │                        │ (Cloud Run)     │
└──────┬───────┘                        └───────┬─────────┘
       ↑                                        │
       │  archive-updates                       │ eval-requests
       │                                        ↓
┌──────┴───────┐                        ┌────────────────┐
│   Archive    │ ←───────────────────── │  Eval Workers   │ (M instances)
│   Updater    │                        │ (Cloud Run)     │
│   (single)   │                        │ (from Phase 1)  │
└──────────────┘                        └─────────────────┘
```

## Component Details

### 1. Dispatcher

**Runs as:** Cloud Run service (single instance, always-on during a search run)
**Responsibility:** Maintain steady parallelism by selecting parents and dispatching mutation requests.

```python
class Dispatcher:
    def __init__(self, algorithm: str, run_id: str, target_parallelism: int):
        self.algorithm = algorithm
        self.run_id = run_id
        self.target_parallelism = target_parallelism
        self.in_flight = 0      # tracked via Pub/Sub message lifecycle
        self.iteration = 0

    def dispatch_batch(self):
        """Called when in_flight < target_parallelism."""
        archive = load_archive_from_firestore(self.run_id)
        search = create_search_algorithm(self.algorithm, archive)

        slots = self.target_parallelism - self.in_flight
        for _ in range(slots):
            self.iteration += 1
            parent = search.select_parent()
            publish_message("mutation-requests", {
                "run_id": self.run_id,
                "iteration": self.iteration,
                "parent_code_uri": parent.gcs_uri,
                "parent_scores": parent.scores,
                "parent_descriptors": parent.descriptors,
                "parent_iteration": parent.iteration,
                "algorithm": self.algorithm,
            })
            self.in_flight += 1

    def on_archive_update(self, message):
        """Called when Archive Updater completes a write."""
        self.in_flight -= 1
        self.dispatch_batch()  # refill the pipeline
```

The Dispatcher calls `select_parent()` — this is the only algorithm-specific logic it needs. Each algorithm's parent selection reads from the Firestore archive:
- **MAP-Elites**: Random occupied cell (`GridArchive.sample()`)
- **Novelty Search**: Weighted by novelty scores (`UnstructuredArchive.sample_weighted_by_novelty()`)
- **Go-Explore**: Weighted by curiosity score (under-explored cells)
- **ADAS**: Top-k designs for meta-agent context

### 2. Mutate Worker

**Runs as:** Cloud Run service (scales 0 to N)
**Triggered by:** Pub/Sub push from `mutation-requests` topic

```python
def handle_mutation_request(message):
    """Cloud Run HTTP handler for mutation requests."""
    parent_code = download_from_gcs(message["parent_code_uri"])
    parent = AgentVariant(
        code=parent_code,
        scores=message["parent_scores"],
        descriptors=message["parent_descriptors"],
        iteration=message["parent_iteration"],
    )

    # Build algorithm-specific mutation context
    search = create_search_algorithm(message["algorithm"])
    new_code, description = search.mutate(parent)

    # Store mutated code in GCS
    code_uri = upload_to_gcs(new_code, message["run_id"], message["iteration"])

    # Publish eval request
    publish_message("eval-requests", {
        "run_id": message["run_id"],
        "iteration": message["iteration"],
        "code_uri": code_uri,
        "description": description,
        "parent_iteration": parent.iteration,
        "algorithm": message["algorithm"],
    })
```

Key design choice: The mutate worker constructs a temporary `OpenEndedSearch` instance to call `mutate()`. This reuses the existing `_build_mutation_prompt()` and `_mutate_openai()` methods from `base.py:298-398` without modification. The algorithm subclass provides `_get_archive_context_for_mutation()` — for MAP-Elites this shows grid occupancy, for ADAS this shows the design archive.

### 3. Eval Worker

**Runs as:** Cloud Run service (from Phase 1 — `spec-eval-fanout.md`)
**Triggered by:** Pub/Sub push from `eval-requests` topic

The eval worker from Phase 1 is reused directly. The only change: after computing scores, it publishes to the `archive-updates` topic instead of returning HTTP response.

```python
def handle_eval_request(message):
    code = download_from_gcs(message["code_uri"])
    scores = run_evaluation_on_code(code)  # existing Phase 1 eval logic
    descriptors = extract_descriptors(code, scores)

    publish_message("archive-updates", {
        "run_id": message["run_id"],
        "iteration": message["iteration"],
        "code_uri": message["code_uri"],
        "scores": scores,
        "descriptors": descriptors,
        "description": message["description"],
        "parent_iteration": message["parent_iteration"],
        "algorithm": message["algorithm"],
    })
```

### 4. Archive Updater

**Runs as:** Cloud Run service (single instance, ordered processing)
**Triggered by:** Pub/Sub push from `archive-updates` topic

```python
def handle_archive_update(message):
    variant = AgentVariant(
        code="",  # not loaded — code_uri is sufficient
        scores=message["scores"],
        descriptors=message["descriptors"],
        iteration=message["iteration"],
        parent_iteration=message["parent_iteration"],
        description=message["description"],
    )
    variant.gcs_uri = message["code_uri"]

    # Load algorithm and archive from Firestore
    search = create_search_algorithm(message["algorithm"])
    archive = load_archive_from_firestore(message["run_id"])

    # Atomic archive update (Firestore transaction)
    accepted = archive.add(variant)

    # Log to BigQuery
    log_to_bigquery(variant, accepted, message["run_id"], message["algorithm"])

    # Notify Dispatcher that a slot is free
    publish_message("dispatcher-notify", {
        "run_id": message["run_id"],
        "accepted": accepted,
    })

    # If best variant improved, update the "best agent" marker in Firestore
    if accepted:
        update_best_agent_marker(message["run_id"], variant)
```

Single-instance processing ensures archive consistency. This is not a bottleneck — archive updates take ~50ms (Firestore transaction + BigQuery insert), while mutations + evals take 30-60 seconds.

## Pub/Sub Topic Configuration

| Topic | Publisher | Subscriber | Message Rate |
|-------|-----------|------------|-------------|
| `mutation-requests` | Dispatcher | Mutate Workers | N msgs/batch |
| `eval-requests` | Mutate Workers | Eval Workers | ~N msgs/minute |
| `archive-updates` | Eval Workers | Archive Updater | ~N msgs/minute |
| `dispatcher-notify` | Archive Updater | Dispatcher | ~N msgs/minute |

All topics use:
- Dead-letter topic after 5 failed delivery attempts
- Message retention: 7 days
- Ack deadline: 600 seconds (matching eval timeout)

## Scaling Knobs

```python
# run_search.py additions
parser.add_argument("--workers", type=int, default=10,
    help="Number of parallel mutation+eval workers")
parser.add_argument("--mode", choices=["local", "distributed"], default="local")
parser.add_argument("--run-id", type=str, help="Unique run identifier")
```

The `--workers` flag sets `target_parallelism` on the Dispatcher. Cloud Run auto-scales Mutate and Eval workers to match demand.

**Practical limits:**
- OpenAI API rate limits (~10K tokens/min on tier 1) are the real ceiling
- With `gpt-4o-mini`, ~20-30 concurrent workers before hitting rate limits
- Can be extended by using multiple API keys or requesting higher rate limits

## Algorithm-Specific Considerations

### MAP-Elites
- `select_parent()` reads a random Firestore cell — fast, no staleness issues
- `update_archive()` compare-and-swap is naturally transactional
- Grid occupancy context for mutations may be slightly stale (worker reads grid, another worker fills a cell before mutation completes) — this is acceptable and even desirable for diversity

### ADAS
- `select_parent()` needs the top-k archive entries for meta-agent context — Firestore query `ORDER BY fitness DESC LIMIT k`
- The meta-agent's "full design history" context grows large — may need pagination or summarization
- Staleness in the design history is acceptable

### Novelty Search
- `select_parent()` uses novelty-weighted sampling — requires computing k-nearest distances across the archive
- This is the most expensive `select_parent()` — may benefit from caching novelty scores in Firestore
- Novelty scores become stale as new variants are added — acceptable, recalculated periodically

### Go-Explore
- `select_parent()` weights by curiosity (visit count inverse) — Firestore counters per cell
- Visit counts are updated by the Archive Updater atomically
- Staleness in visit counts means some cells get explored slightly more than intended — acceptable

## Files to Create/Modify

| File | Action | Description |
|------|--------|-------------|
| `cloud/dispatcher.py` | Create | Dispatcher service |
| `cloud/mutate_worker.py` | Create | Mutation worker service |
| `cloud/archive_updater.py` | Create | Archive update service |
| `cloud/pubsub.py` | Create | Pub/Sub publish/subscribe helpers |
| `cloud/deploy-workers.sh` | Create | Deployment script for all services |
| `cloud/pubsub-setup.sh` | Create | Topic/subscription creation script |
| `algorithms/base.py` | Modify | Extract `mutate()` context-building into reusable helper |
| `run_search.py` | Modify | Add `--mode distributed`, `--workers`, `--run-id` flags |

## Verification

1. Deploy all services: eval (Phase 1), archive (Phase 2), dispatcher, mutate workers, archive updater
2. Run: `python run_search.py map-elites --mode distributed --workers 10 --run-id test-dist-1 --max-iterations 50`
3. Confirm:
   - 10 mutations running concurrently (check Cloud Run metrics)
   - Archive in Firestore is populated with variants
   - BigQuery shows ~50 rows for `run_id = "test-dist-1"`
   - No lost updates (archive cell fitness is monotonically non-decreasing)
   - Throughput is ~8-10x higher than single-worker mode
4. Kill the dispatcher mid-run, restart — confirm it resumes from Firestore state
5. Run two algorithms simultaneously on different `run_id`s — confirm no interference
