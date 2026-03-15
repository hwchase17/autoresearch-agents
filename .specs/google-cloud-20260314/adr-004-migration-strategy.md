# ADR-004: Incremental Migration Strategy

**Status:** Proposed
**Date:** 2026-03-14

## Context

The project currently has zero cloud infrastructure:
- No Dockerfile
- No CI/CD
- No `requirements.txt` or `pyproject.toml`
- Dependencies installed manually via `pip install`
- Git used only for tracking agent.py mutations, not for deployment

A big-bang migration to the full distributed architecture (eval fan-out + distributed archive + parallel search workers + Pub/Sub orchestration) is high risk and delays the first usable improvement.

## Decision

**Migrate in three phases. Each phase delivers independent value and can be shipped alone.**

### Phase 1: Parallel Eval (Cloud Run)

**What changes:**
- Containerize `run_eval.py` + `agent.py` + dependencies as a Cloud Run service
- Add a `CloudEvalClient` class that `base.py:_run_eval()` can call instead of `subprocess.run()`
- `run_search.py` gains `--eval-mode local|cloud` flag
- Add `Dockerfile`, `requirements.txt`, and deploy script

**What stays the same:**
- Search loop still runs locally (`while True` in `base.py`)
- Archive state still in local JSON files
- Results still logged to local TSV
- Single search algorithm at a time

**Value delivered:**
- Eval time drops from ~5 min → ~30 sec per iteration (20x improvement)
- For a 50-iteration MAP-Elites run: ~4 hours → ~25 minutes
- Local machine just orchestrates — can be a laptop

**Effort:** Small. One Dockerfile, one new class, one CLI flag.

### Phase 2: Distributed State (Firestore + GCS + BigQuery)

**What changes:**
- `GridArchive` and `UnstructuredArchive` gain Firestore-backed implementations
- `AgentVariant.code` moves to GCS; dataclass gets `gcs_uri` field
- `_log_result()` writes to BigQuery instead of TSV
- `save_state()`/`load_state()` use Firestore instead of JSON files
- Local fallback preserved: `--storage local|cloud` flag

**What stays the same:**
- Search loop still runs locally (but now reads/writes cloud state)
- Eval uses Cloud Run (from Phase 1)
- Single search algorithm at a time

**Value delivered:**
- State survives machine failure — can stop and resume from any machine
- Results are queryable in BigQuery
- Foundation for Phase 3 (parallel workers need shared state)

**Effort:** Medium. Archive classes need cloud-aware implementations with Firestore transactions.

### Phase 3: Parallel Search Workers (Pub/Sub + Cloud Run Jobs)

**What changes:**
- `OpenEndedSearch.run()` loop decomposed into Dispatcher + Workers + Archive Updater (see ADR-003)
- Pub/Sub topics for mutation-requests, eval-requests, archive-updates
- Workers deployed as Cloud Run services
- `run_search.py` gains `--workers N` and `--mode local|distributed` flags

**What stays the same:**
- Algorithm logic (`select_parent`, `update_archive`, mutation prompts) unchanged
- Eval service from Phase 1, storage from Phase 2

**Value delivered:**
- N workers = ~Nx throughput (limited by API rate limits)
- Multiple algorithms can run simultaneously on the same or different archives
- Full hands-off operation: start a search, check BigQuery later

**Effort:** Large. Requires the Pub/Sub pipeline, worker services, dispatcher logic, and operational tooling.

## Alternatives Considered

### Big-bang migration (all three phases at once)

Pros: Architecturally clean — no intermediate states to maintain.
Cons: Weeks of work before any value is delivered. Higher risk of integration issues. Can't validate assumptions (e.g., "is Cloud Run cold start acceptable?") incrementally.

**Rejected because**: Phase 1 alone delivers 20x eval speedup with minimal effort. There's no reason to wait for Phases 2-3 to capture that value.

### Skip Phase 2 (go from local state to distributed workers)

Pros: Fewer intermediate steps.
Cons: Parallel workers with local file state is impossible — workers can't share a JSON file across machines. Phase 2 is a prerequisite for Phase 3.

**Rejected because**: It's not a valid option. Distributed workers require distributed state.

### Start with Phase 3 (distributed workers first, optimize eval later)

Pros: Gets to the "many parallel experiments" vision faster.
Cons: Each worker still waits 5 minutes for sequential eval. 10 workers doing 5-minute evals is better than 1 worker, but 10 workers doing 30-second evals is dramatically better.

**Rejected because**: Phase 1 is the highest-ROI change and is a prerequisite for Phase 3 workers to be efficient.

## Consequences

### Positive
- Each phase is independently valuable and shippable
- Phase 1 can be completed quickly and validates the containerization approach
- Local development mode preserved at every phase (`--mode local` flags)
- Risk is spread across three smaller deliveries instead of one large one

### Negative
- Maintaining dual code paths (local vs cloud) adds complexity to `base.py` and archive classes
- Three deployment steps means three rounds of testing and validation
- Phase 2 may feel like "infrastructure work" with less visible user impact than Phases 1 and 3

### Verification Criteria Per Phase

**Phase 1:** `python run_search.py map-elites --eval-mode cloud --max-iterations 5` completes in <5 minutes (vs ~25 minutes with local eval)

**Phase 2:** Stop a search mid-run, restart on a different machine, confirm it resumes from the correct iteration with full archive state

**Phase 3:** `python run_search.py map-elites --mode distributed --workers 10 --max-iterations 50` produces ~10x more iterations/hour than single-worker mode
