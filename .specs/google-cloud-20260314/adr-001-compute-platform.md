# ADR-001: Compute Platform for Eval and Search Workers

**Status:** Proposed
**Date:** 2026-03-14

## Context

The system has two parallelism opportunities:

1. **Eval fan-out**: `run_eval.py` evaluates 20 test cases at `max_concurrency=4`. Each case calls the OpenAI API (agent inference) plus an LLM-as-judge call. These are independent and could run fully in parallel.

2. **Search worker fan-out**: The `OpenEndedSearch.run()` loop in `algorithms/base.py:84` is sequential — one `select_parent → mutate → evaluate → update_archive` cycle at a time. Population-based algorithms can run many of these cycles concurrently.

Both workloads are I/O-bound (waiting on OpenAI API responses), require minimal CPU/RAM, and are stateless per invocation. The question is which GCP compute primitive fits each.

## Decision

**Eval fan-out: Cloud Run**

Deploy the evaluation harness as a Cloud Run service. Each invocation accepts an agent code blob + single test case, runs `run_agent_for_eval()` from `run_eval.py`, executes the three evaluators (correctness, helpfulness, tool_usage), and returns scores.

- Container holds the fixed dependencies (`langsmith`, `langchain-openai`, `langgraph`, `pydantic`)
- Scales to 0 when idle — no cost between experiments
- Concurrency limit of 1 per instance (each eval case gets its own container) to avoid module-level state collisions in `agent.py`
- Max instances capped at 20 (one per dataset example) to control API spend

**Search workers: Cloud Run Jobs**

Each search iteration (select, mutate, evaluate, propose archive update) runs as a Cloud Run Job task. A coordinator dispatches N tasks in parallel, each producing a candidate `AgentVariant`. The coordinator then applies archive updates sequentially (see ADR-003).

- Jobs are better than services here because each iteration is a batch unit of work, not a request/response
- Tasks auto-retry on transient failures
- Parallelism controlled by `--tasks` flag (e.g., 10 concurrent mutations per generation)

## Alternatives Considered

### Cloud Functions (Gen 2)

Pros: Finest-grained scaling, sub-second cold starts for Python, simpler deploy than containers.
Cons: 10-minute timeout may be tight for eval + judge calls on slow API days. No native "job" concept — would need Cloud Tasks as a wrapper for search iterations. Memory limit (32 GB) is more than enough but the execution model encourages very small units of work, leading to more orchestration complexity.

**Rejected because**: The eval harness imports `langchain_openai`, `langgraph`, and `langsmith`, which have non-trivial import times. Cloud Run's container model handles this better (warm instances). The 10-minute timeout is also a real risk — `eval_timeout` defaults to 600s in `base.py`.

### GKE (Google Kubernetes Engine)

Pros: Full control over scheduling, persistent pods, sidecar patterns.
Cons: Massive operational overhead for this workload. Always-on node pools mean paying for idle compute. The workload is bursty (run 50 iterations, then nothing for hours) — GKE's autoscaling is slower than Cloud Run's.

**Rejected because**: This is not a long-running service. It's batch-style, bursty, I/O-bound work. GKE is over-provisioned for the problem.

### Compute Engine (raw VMs)

Pros: Maximum flexibility.
Cons: Manual scaling, manual fault tolerance, no container isolation.

**Rejected because**: Reimplements what Cloud Run provides out of the box.

## Consequences

### Positive
- Eval time drops from ~5 minutes (20 cases / 4 concurrent) to ~30 seconds (20 cases fully parallel)
- Search throughput scales linearly with worker count — 10 workers = ~10x iterations/hour
- Zero cost when not running experiments
- Each worker is isolated — a crash in one mutation doesn't affect others

### Negative
- Requires containerizing the evaluation harness (Dockerfile for `run_eval.py` + dependencies)
- Cold start latency (~2-5s) adds overhead per eval invocation — mitigated by min-instances=1 during active experiments
- API key management moves from local env vars to Secret Manager
- Network latency between Cloud Run and OpenAI API adds ~50ms per call (negligible vs API response time)

### Impact on Existing Code
- `base.py:_run_eval()` — currently shells out via `subprocess.run("python run_eval.py")`. In cloud mode, this becomes an HTTP call to the Cloud Run eval service
- `run_eval.py:run_evaluation()` — the `evaluate()` call with `max_concurrency=4` is replaced by the Cloud Run service handling concurrency natively
- `run_search.py` — gains a `--cloud` flag to switch between local and cloud eval
