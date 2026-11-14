# Closed-Loop Elastic Multi-Agent Swarm Blueprint (Domain, Harness & Tenant Agnostic)

> **Purpose:** A reusable engineering blueprint for solving complex, high-precision tasks using an **elastic multi-agent swarm (`--num-workers 1..100+`)** with **deterministic reward control**, **zero-hallucination human-in-the-loop gates**, **enterprise tenant API support**, and **live Mission Control telemetry**.
>
> **Portability:** Works with any AI coding harness (Claude Code, Codex, Cursor, Gemini CLI, OpenCode, Antigravity), any LLM endpoint (internal tenant gateway, Azure OpenAI, Vertex, Bedrock, vLLM/LiteLLM, or public APIs), and any orchestration framework.

---

## 1. Architectural Overview

Instead of a small sequential crew, structure the system as a **parameterized, 4-stage closed-loop elastic swarm**:

| Stage | Name | Execution | Objective |
| :--- | :--- | :--- | :--- |
| **Phase 0** | Deterministic Baseline | `0 tokens` | Parse raw inputs, run rule-based checks, compute initial score $S_0 \in [0, 100]$ and residual $e(0) = 100 - S_0$. |
| **Phase 1** | Elastic Closed-Loop Convergence ($k=1 \dots N$) | Parallel Pool (`--num-workers`) | Scale to 20–100+ workers (`sharded` or `tournament` mode) to reduce residual $e(k)$ until $e(k) \le \varepsilon$ or `--max-iterations` is reached. |
| **Gate** | Human-in-the-Loop (HITL) | `0 tokens` | Collect explicit user confirmation for every `[CONFIRM WITH USER]` gap before compiling final deliverables. |
| **Phase 2** | Deep Scout & Deterministic Ranking *(Optional)* | Parallel Search + HTTP | Fan out across N targets in parallel, verify live HTTP reachability, and rank options using a deterministic weighted formula. |

---

## 2. Parameterized Swarm & Tenant Configuration

Expose swarm scale and tenant routing as first-class CLI flags and `.env` variables:

```bash
# Example: 100 logical workers, 20 concurrent calls against internal tenant gateway
./run.sh run \
  --num-workers 100 \
  --max-concurrency 20 \
  --swarm-mode tournament \
  --top-k 5 \
  --max-iterations 10
```

### Key Input Variables
| Variable / Flag | Default | Purpose |
| :--- | :--- | :--- |
| `--num-workers` / `NUM_WORKERS` | `20` (supports `1..100+`) | Logical number of execution agents per stage/epoch. |
| `--max-concurrency` / `MAX_CONCURRENCY` | `10` | Real in-flight API calls bounded by `asyncio.Semaphore` / `ThreadPoolExecutor` to respect tenant TPM/RPM quotas. |
| `--swarm-mode` | `sharded` \| `tournament` | **`sharded`**: Map-Reduce across $N$ data slices.<br>**`tournament`**: Best-of-$N$ diverse proposals on the same hard task, keeping `--top-k` winners. |
| `--top-k` | `5` | Number of winning candidates promoted by the Deterministic Judge in `tournament` mode. |
| `--budget-usd` | `0` (unlimited / telemetry-only) | Optional soft chargeback/governance cap; disable hard stops when running on flat/internal tenant quotas. |
| `LLM_BASE_URL` / `OPENAI_BASE_URL` | *(tenant endpoint)* | Internal API gateway, LiteLLM proxy, Azure/Vertex/Bedrock endpoint. |
| `LLM_TIER1_MODEL` / `LLM_TIER2_MODEL` | *(tenant deployments)* | Primary model and escalation model (can default directly to Tier 2 when cost is not a constraint). |

---

## 3. Standard Directory Layout

```text
project_root/
├── AGENTS.md                  # Harness-agnostic project rules & commands
├── .env.example               # Tenant base URL, API keys, NUM_WORKERS, MAX_CONCURRENCY
├── requirements.txt           # Minimal pinned dependencies
├── run.sh / run.bat           # Interactive + CLI runner forwarding swarm flags
├── input/                     # Raw inputs + user_confirmations.json
├── output/                    # Generated artifacts + telemetry_live.json
└── src/
    ├── main.py                # CLI entrypoint (--stage, --num-workers, --max-concurrency, --swarm-mode)
    ├── config/                # Domain dimensions, rubric weights, persona/temperature pool
    ├── domain/                # Domain-specific parser, validator, artifact compiler
    ├── core/
    │   ├── swarm_runner.py    # Bounded async/thread pool + 429 Retry-After backoff + Map-Reduce/Tournament
    │   ├── reward_engine.py   # Deterministic scoring + hybrid critic fusion
    │   ├── escalation.py      # Tier 1 -> Tier 2 model escalation policy
    │   ├── telemetry.py       # Thread-safe single-writer queue/lock for telemetry_live.json
    │   ├── web_scout.py       # Search API + HTTP 200..399 link verifier
    │   └── notifier.py        # Optional milestone alerts (WhatsApp / Slack / Teams / Webhook)
    └── dashboard/
        ├── server.py          # Lightweight local HTTP server (default 127.0.0.1:8765)
        ├── index.html         # SPA rendering aggregated worker_pools + convergence curve
        └── confirmations.html # Local web form posting to /api/confirmations
```

---

## 4. Scaling Safely to 20–100+ Agents

### 4.1. Bounded Concurrency & Tenant Quota Protection
Even when cost is negligible on an internal tenant, **TPM/RPM quotas and gateway timeouts** still apply:
- Decouple `--num-workers` (e.g., `100`) from `--max-concurrency` (e.g., `15–25`).
- Wrap every LLM/HTTP call in `Semaphore(max_concurrency)` + exponential backoff with jitter honoring `Retry-After` headers on `429` / `503`.

### 4.2. Shard-then-Reduce & Tournament Selection (No Shared-State Races)
- **Never** let 100 workers mutate the final artifact concurrently.
- Each worker receives an immutable input snapshot + unique `worker_id` and returns an isolated result object.
- **In `sharded` mode:** A single **Reducer** merges all $N$ validated shards deterministically.
- **In `tournament` mode:** All $N$ workers propose competing solutions (varying temperature/prompt angle); the **Deterministic Reward Engine** scores all $N$ and promotes only `--top-k` to the Critic / next epoch.

### 4.3. Thread-Safe Single-Writer Telemetry
- 100 workers writing `output/telemetry_live.json` simultaneously will corrupt JSON or lose updates.
- Route all worker status/metrics through a **single thread-safe lock or aggregator queue** (`write temp -> os.replace`).
- Store **aggregated pool metrics** (`worker_pools`) instead of 100 raw UI cards so the dashboard stays fast and readable.

### 4.4. Zero-Hallucination & Human-in-the-Loop (HITL) Protocol
- **Hard Rule:** Never fabricate dates, identifiers, metrics, citations, URLs, or credentials.
- Any missing or ambiguous fact must be emitted verbatim as `[CONFIRM WITH USER: <specific question>]`.
- Aggregate all pending items into `input/user_confirmations.json` and expose them via `confirmations.html` (`GET /api/confirmations`, `POST /api/confirmations`).

---

## 5. Telemetry Contract (`output/telemetry_live.json`)

```json
{
  "meta": {
    "run_mode": "phase1_elastic_swarm",
    "swarm_mode": "tournament",
    "num_workers": 100,
    "max_concurrency": 20,
    "status": "RUNNING | COMPLETED | FAILED",
    "current_stage": "Epoch k=2: Evaluating 100 workers (Top-5 selection)",
    "current_iteration": 2,
    "max_iterations": 10,
    "target_score": 96.0,
    "target_residual": 4.0
  },
  "kpis": {
    "initial_score": 49.0,
    "current_score": 88.2,
    "initial_residual": 51.0,
    "current_residual": 11.8,
    "residual_reduction_pct": 76.9,
    "current_tier": "Gold",
    "total_tokens": 245000,
    "total_cost_usd": 0.0,
    "model_escalations": 1,
    "elapsed_seconds": 24.7
  },
  "worker_pools": {
    "execution_pool": {
      "label": "Execution Workers",
      "total": 100,
      "queued": 20,
      "running": 20,
      "completed": 59,
      "failed": 1,
      "best_score": 91.4,
      "avg_latency_s": 2.3
    }
  },
  "dimension_residuals": {
    "dim_1": { "pontos": 14.5, "maximo": 15.0, "residuo": 0.5, "residuo_inicial": 13.0 }
  },
  "convergence_series": [
    { "iteration": 0, "overall_score": 49.0, "residual": 51.0, "delta_residual": 0.0 }
  ],
  "agent_messages": [
    { "id": "msg_1", "ts": "12:01:04", "iteration": 2, "sender_id": "worker_042", "recipient_id": "judge", "msg_type": "TOP_K_WINNER", "summary": "...", "content": "..." }
  ],
  "events_log": [
    { "ts": "12:01:04", "tag": "iter_2", "message": "Pool 100/100 completed; Top-5 promoted." }
  ]
}
```

---

## 6. Harness Portability Checklist

1. **No Hardcoded Paths:** Resolve all paths relative to project root.
2. **Tenant Endpoint Ready:** Configure `LLM_BASE_URL` and API credentials via `.env` only; never hardcode provider URLs.
3. **Respect Sandbox Boundaries:** Separate offline Phase 0 / Dashboard from network-dependent Phase 1 / Phase 2 runs.
4. **Zero-Dependency UI:** Keep the dashboard as pure static HTML/JS served by Python's built-in `http.server` so Node.js/npm is never required.
