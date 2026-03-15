# Spec: Parallel Evaluation Service

**Phase:** 1 (highest ROI, no dependencies on other phases)
**Related ADRs:** [ADR-001](adr-001-compute-platform.md), [ADR-004](adr-004-migration-strategy.md)

## Overview

A Cloud Run service that accepts an agent implementation + evaluation dataset and runs all test cases in parallel, returning aggregated scores. Replaces the local `subprocess.run("python run_eval.py")` call in `algorithms/base.py:184-208`.

## Why This Is Phase 1

The current eval pipeline is the single largest time bottleneck. Each iteration of any search algorithm blocks on `_run_eval()`, which:

1. Shells out to `python run_eval.py` (`base.py:188`)
2. Runs `evaluate()` from LangSmith with `max_concurrency=4` (`run_eval.py:218-224`)
3. Iterates 20 test cases, each involving: agent inference (OpenAI API) + up to 3 LLM-as-judge calls (OpenAI API)

At `max_concurrency=4`, a full eval takes ~5 minutes. At full parallelism (20 concurrent), it drops to ~30 seconds. This 10x improvement applies to every single iteration of every search algorithm.

## Current Interface

```python
# algorithms/base.py:184
def _run_eval(self) -> dict:
    result = subprocess.run(
        self.eval_cmd.split(),  # "python run_eval.py"
        capture_output=True, text=True,
        timeout=self.eval_timeout,
        cwd=self.agent_path.parent,
    )
    # ... parse stdout for scores dict
    return self._parse_eval_output(output)
```

Returns a dict like:
```python
{
    "overall_score": 0.833,
    "avg_correctness": 0.85,
    "avg_helpfulness": 0.90,
    "avg_tool_usage": 0.75,
    "num_examples": 20,
    "num_errors": 0,
    "experiment_url": "https://smith.langchain.com/..."
}
```

## Proposed Cloud Service

### Endpoint

```
POST /evaluate
Content-Type: application/json

{
    "agent_code": "<full agent.py source code>",
    "dataset_name": "autoresearch-agent-eval",
    "experiment_prefix": "autoresearch",
    "run_id": "map-elites-run-42",
    "iteration": 17
}
```

### Response

```json
{
    "overall_score": 0.833,
    "avg_correctness": 0.85,
    "avg_helpfulness": 0.90,
    "avg_tool_usage": 0.75,
    "num_examples": 20,
    "num_errors": 0,
    "experiment_url": "https://smith.langchain.com/..."
}
```

Identical schema to what `_parse_eval_output()` currently returns. This is critical — the calling code in `base.py` should not need to change its score-handling logic.

### How It Works Internally

1. Receive request with agent code
2. Write agent code to a temp file within the container
3. Call `run_evaluation()` from `run_eval.py` with `max_concurrency=20` (all cases in parallel)
4. Return the summary dict as JSON

The service container bundles:
- `run_eval.py` (the fixed evaluation harness)
- `dataset.json` (baked into the container image, or fetched from GCS)
- All dependencies: `langsmith`, `langchain-openai`, `langgraph`, `pydantic`

### Container Configuration

```yaml
# Cloud Run service config
service: autoresearch-eval
region: us-central1
memory: 512Mi          # Minimal — all heavy compute is on OpenAI's side
cpu: 1
timeout: 600s          # Match current eval_timeout default
max-instances: 5       # Limit concurrent eval runs to control API spend
concurrency: 1         # One eval per container (agent.py module-level state)
min-instances: 0       # Scale to zero between experiments
```

### Environment Variables (via Secret Manager)

```
OPENAI_API_KEY        — for agent inference + judge calls
LANGSMITH_API_KEY     — for tracing
LANGSMITH_TRACING=true
```

## Local Client Integration

New class in `algorithms/base.py` (or a new `algorithms/cloud_eval.py`):

```python
class CloudEvalClient:
    """Calls the Cloud Run eval service instead of local subprocess."""

    def __init__(self, service_url: str, timeout: int = 600):
        self.service_url = service_url
        self.timeout = timeout

    def evaluate(self, code: str, run_id: str, iteration: int) -> dict:
        """Send agent code to cloud eval and return scores dict."""
        import requests
        resp = requests.post(
            f"{self.service_url}/evaluate",
            json={
                "agent_code": code,
                "run_id": run_id,
                "iteration": iteration,
            },
            timeout=self.timeout,
        )
        resp.raise_for_status()
        return resp.json()
```

The `OpenEndedSearch._run_eval()` method switches between local and cloud:

```python
def _run_eval(self) -> dict:
    if self.cloud_eval_client:
        return self.cloud_eval_client.evaluate(
            self.agent_path.read_text(),
            run_id=self.run_id,
            iteration=self.iteration,
        )
    else:
        # existing subprocess.run logic
        ...
```

## Error Handling

| Failure | Behavior |
|---------|----------|
| Cloud Run timeout (600s) | Return `{"overall_score": 0.0, "timed_out": True}` — same as current local behavior |
| OpenAI API rate limit | Retry with backoff within the container (LangSmith's `evaluate()` handles this) |
| Container crash | Cloud Run auto-restarts; client gets HTTP 500, retries once |
| Network error (client → Cloud Run) | Client retries up to 3 times with exponential backoff |
| Partial eval failure (some cases error) | Return scores from successful cases + `num_errors` count — same as current behavior in `run_eval.py:236-238` |

## LangSmith Integration

LangSmith tracing works identically in the container — the `LANGSMITH_API_KEY` and `LANGSMITH_TRACING` env vars are set, and `langsmith.evaluate()` in `run_eval.py` handles trace creation. The `experiment_prefix` and dataset name are passed through. No changes needed to the tracing logic.

## Deployment

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/$PROJECT/autoresearch-eval
gcloud run deploy autoresearch-eval \
    --image gcr.io/$PROJECT/autoresearch-eval \
    --region us-central1 \
    --memory 512Mi \
    --timeout 600 \
    --concurrency 1 \
    --max-instances 5 \
    --set-secrets OPENAI_API_KEY=openai-key:latest,LANGSMITH_API_KEY=langsmith-key:latest \
    --set-env-vars LANGSMITH_TRACING=true
```

## Files to Create/Modify

| File | Action | Description |
|------|--------|-------------|
| `Dockerfile` | Create | Container for eval service |
| `requirements.txt` | Create | Pin dependencies for reproducible builds |
| `algorithms/cloud_eval.py` | Create | `CloudEvalClient` class |
| `algorithms/base.py` | Modify | Add `cloud_eval_client` to `__init__`, switch in `_run_eval()` |
| `run_search.py` | Modify | Add `--eval-mode local\|cloud` and `--eval-service-url` flags |
| `cloud/deploy-eval.sh` | Create | Deployment script |

## Verification

1. Deploy the eval service to Cloud Run
2. Run `python run_search.py map-elites --eval-mode cloud --eval-service-url $URL --max-iterations 3`
3. Confirm each iteration completes in <1 minute (vs ~5 minutes locally)
4. Confirm scores match a local eval run on the same agent code
5. Confirm LangSmith traces appear for cloud eval runs
