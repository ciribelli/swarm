# AGENTS.md (Template for New Elastic Swarm Projects)

> Drop this file at the root of any new project scaffolded from `SWARM_BLUEPRINT.md` so any AI harness immediately knows how to operate safely and consistently.

## 1. Project Identity & Goal
- **Domain:** `[Fill in domain]`
- **Architecture:** Closed-Loop Elastic Multi-Agent Swarm (`SWARM_BLUEPRINT.md`).
- **Python Environment:** Always use `.venv/bin/python` (macOS/Linux) or `.venv\Scripts\python.exe` (Windows).

## 2. Standard Commands
- **Start Live Dashboard:** `./run.sh dashboard` (serves `http://127.0.0.1:8765`)
- **Run Phase 0 (Deterministic Baseline, 0 tokens):** `./run.sh phase0`
- **Run Convergence Simulation:** `./run.sh simulate --num-workers 20`
- **Run Phase 1 (Elastic AI Swarm):** `./run.sh run --num-workers 50 --max-concurrency 15 --swarm-mode tournament --top-k 5`
- **Run Phase 2 (Parallel Deep Scout):** `./run.sh scout --max-concurrency 15`

## 3. Immutable Guardrails (Zero Hallucination)
1. **Never fabricate** dates, metrics, IDs, citations, or URLs.
2. Any unverified or missing required field **must** be marked `[CONFIRM WITH USER: <question>]` and reconciled via `input/user_confirmations.json` before final export.
3. **Domain Hard Constraints:**
   - `[Add immutable facts, forbidden names/terms, or compliance rules here]`

## 4. Concurrency, Tenant & Sandbox Notes
- **Tenant Routing:** All LLM calls must read `LLM_BASE_URL` / `OPENAI_BASE_URL` and keys from `.env` (never hardcode endpoints).
- **Concurrency Safety:** Always decouple `--num-workers` from `--max-concurrency` (use bounded semaphore + `429 Retry-After` backoff).
- **Single-Writer Telemetry:** All worker pool updates to `output/telemetry_live.json` must pass through the thread-safe telemetry lock/queue (`write temp -> os.replace`).
- **Network Access:** Phase 0 and the local dashboard work offline inside a sandbox. Phase 1 (tenant LLM API calls) and Phase 2 (search + HTTP verification) require outbound/intranet network access.
- **No Absolute Paths:** Never hardcode machine-specific home directories in `src/`, `run.sh`, or `run.bat`.
