# GCP Parallelized Workers Proposal

**Date:** 2026-03-14
**Status:** Proposed
**Author:** Claude Code

## Motivation

The autoresearch-agents project runs all computation serially on a single machine. The population-based search algorithms (MAP-Elites, ADAS, Novelty Search, Go-Explore) are embarrassingly parallel — each iteration independently selects a parent, mutates it, evaluates the result, and proposes an archive update — yet the current `while True` loop in `algorithms/base.py` processes one iteration at a time.

The evaluation harness (`run_eval.py`) further bottlenecks throughput: 20 test cases run at `max_concurrency=4`, and a single eval pass must complete before the next mutation begins.

All inference is already API-driven (OpenAI, LangSmith). The local machine contributes only orchestration and string parsing. GCP deployment unlocks **throughput** (more experiments per hour) and **reliability** (crash recovery, persistent state), not raw compute power. No GPUs are needed.

## What This Proposal Covers

### Architecture Decision Records

| ADR | Decision |
|-----|----------|
| [ADR-001](adr-001-compute-platform.md) | Compute platform for eval workers and search workers |
| [ADR-002](adr-002-state-and-storage.md) | Persistent state backend (archive, results, variants) |
| [ADR-003](adr-003-orchestration.md) | Pipeline orchestration and worker coordination |
| [ADR-004](adr-004-migration-strategy.md) | Incremental migration path (3 phases) |

### Component Specifications

| Spec | Component |
|------|-----------|
| [spec-eval-fanout](spec-eval-fanout.md) | Parallel evaluation service (Phase 1, highest ROI) |
| [spec-distributed-archive](spec-distributed-archive.md) | Cloud-backed archive with concurrency control |
| [spec-search-worker](spec-search-worker.md) | Distributed search algorithm execution |

## Key Principle

The cost is dominated by OpenAI/LangSmith API calls, not GCP infrastructure. GCP buys throughput and resilience. The migration is phased so each step delivers independent value.
