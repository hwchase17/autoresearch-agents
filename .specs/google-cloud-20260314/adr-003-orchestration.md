# ADR-003: Pipeline Orchestration and Worker Coordination

**Status:** Proposed
**Date:** 2026-03-14

## Context

The current "orchestration" is a `while True` loop in `algorithms/base.py:84-146`:

```python
while True:
    parent = self.select_parent()       # read from archive
    new_code, description = self.mutate(parent)  # LLM call
    scores, descriptors = self.evaluate(new_code) # shell out to run_eval.py
    accepted = self.update_archive(variant)       # write to archive
    self.save_state()
```

Parallelizing this loop means multiple workers execute the `select → mutate → evaluate` pipeline concurrently, then compete to `update_archive`. The coordination challenge is:

1. **Dispatch**: How do workers know when to start a new iteration?
2. **Synchronization**: `update_archive` is the critical section — two workers filling the same grid cell in MAP-Elites must not lose data.
3. **Completion**: How does the system know when to stop?
4. **Failure handling**: What happens when a worker crashes mid-mutation or mid-eval?

## Decision

**Pub/Sub + Cloud Run for event-driven coordination.**

The pipeline is decomposed into three stages connected by Pub/Sub topics:

```
[Dispatcher] → topic: mutation-requests
                  ↓
            [Mutate Worker] (Cloud Run) → topic: eval-requests
                                            ↓
                                      [Eval Worker] (Cloud Run) → topic: archive-updates
                                                                     ↓
                                                               [Archive Updater] (single instance)
                                                                     ↓
                                                               [Dispatcher] (loops back)
```

### Components

**Dispatcher** (Cloud Run, single instance)
- Maintains the target parallelism level (e.g., 10 concurrent mutations)
- Publishes `mutation-request` messages containing: `run_id`, `iteration`, `parent_variant_ref` (Firestore doc ID), `algorithm` type
- Uses `select_parent()` to choose parents — this is a fast read from Firestore, safe to centralize
- Publishes new requests as archive-update confirmations arrive (maintains a steady pipeline)

**Mutate Worker** (Cloud Run, scales 0-N)
- Receives a `mutation-request` via Pub/Sub push subscription
- Loads parent code from GCS, calls `mutate()` (LLM API call)
- Writes new code to GCS, publishes `eval-request` with the GCS URI
- Stateless — no local archive access needed

**Eval Worker** (Cloud Run, scales 0-N)
- Receives an `eval-request` via Pub/Sub push subscription
- Loads agent code from GCS, runs evaluation (20 test cases in parallel within the container)
- Publishes `archive-update` with scores, descriptors, and variant metadata
- This is the same service described in `spec-eval-fanout.md`

**Archive Updater** (Cloud Run, single instance, ordered processing)
- Receives `archive-update` messages
- Calls `update_archive(variant)` within a Firestore transaction
- Logs result to BigQuery
- Acks message only after successful archive update
- Single instance ensures archive consistency without distributed locking

### Why Single Archive Updater?

The `update_archive` operation in MAP-Elites is a compare-and-swap on a grid cell. In ADAS/Novelty/Go-Explore, it's an append + possible eviction. Making this a single-writer bottleneck is intentional:

- Archive updates are fast (~10ms Firestore write) — not a throughput bottleneck
- Eliminates all distributed locking complexity
- The expensive work (mutation + evaluation) is fully parallel
- Even with 50 parallel workers, the updater handles the throughput easily

## Alternatives Considered

### Cloud Workflows

Pros: Declarative pipeline definition, built-in retry logic, visual execution graph.
Cons: Workflows orchestrate sequential steps well but don't natively support "run N parallel iterations and loop." Each workflow execution is a single pipeline run — you'd need an outer loop to dispatch new iterations, which becomes a custom dispatcher anyway.

**Rejected because**: The loop-and-dispatch pattern doesn't map cleanly to Workflows' step-based model. We'd end up building custom coordination on top of Workflows, gaining little over Pub/Sub.

### Cloud Tasks

Pros: At-least-once delivery, rate limiting, scheduled dispatch.
Cons: Cloud Tasks is a task queue — tasks are pushed to a target HTTP endpoint. This is functionally similar to Pub/Sub push but with less flexibility (no fan-out, no filtering, no dead-letter topics).

**Rejected because**: Pub/Sub provides the same push delivery model with better fan-out semantics and dead-letter handling. Cloud Tasks would work but offers no advantage here.

### Direct HTTP orchestration (coordinator calls workers synchronously)

Pros: Simplest mental model — coordinator fires N HTTP requests, awaits responses, updates archive.
Cons: The coordinator must hold N concurrent connections open. If any worker takes 10 minutes (eval timeout), the coordinator's request also times out. No automatic retry. Tight coupling between coordinator and worker lifecycles.

**Rejected because**: Too fragile for long-running eval operations. Pub/Sub decouples producer and consumer lifecycles.

## Consequences

### Positive
- Mutation and evaluation run fully in parallel (limited only by API rate limits and configured max workers)
- Each stage is independently scalable — can have 5 mutate workers and 20 eval workers if mutation is the bottleneck
- Dead-letter topics catch failed messages for debugging
- The dispatcher controls pacing — can throttle to stay within API budget
- Graceful degradation: if eval workers are slow, mutation requests queue up rather than failing

### Negative
- Three Pub/Sub topics to manage (mutation-requests, eval-requests, archive-updates)
- Message ordering is not guaranteed — archive updates arrive in arbitrary order (acceptable: all algorithms are robust to this)
- End-to-end latency for a single iteration increases slightly (Pub/Sub propagation ~100ms per hop)
- Debugging requires correlating logs across 4 services + 3 topics (mitigated by `run_id` trace field)
- Pub/Sub costs: free tier covers 10 GB/month, well within our message volume

### Impact on Existing Code
- `base.py:OpenEndedSearch.run()` — the while-loop is replaced by the Dispatcher + Archive Updater pattern
- `base.py:mutate()` — extracted into the Mutate Worker service
- `base.py:evaluate()` — extracted into the Eval Worker service (see spec-eval-fanout.md)
- `base.py:select_parent()` — called by the Dispatcher, reads from Firestore archive
- `base.py:update_archive()` — called by the Archive Updater, writes to Firestore
- Each algorithm subclass (`map_elites.py`, etc.) keeps its `select_parent`/`update_archive` logic but the calling context changes from a local loop to message handlers
