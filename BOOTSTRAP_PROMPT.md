# Master Bootstrap Prompt (Copy & Paste into Any AI Harness)

Use the prompt below when starting a **brand-new project** on another computer with any AI coding harness (Claude Code, Codex, Cursor, Gemini CLI, OpenCode, Antigravity, etc.).

---

```text
Read `SWARM_BLUEPRINT.md` in this workspace carefully. We are going to apply this exact Closed-Loop Elastic Multi-Agent Swarm architecture to a NEW domain, running against our internal API tenant.

### My New Problem / Domain
- **Domain:** [DESCRIBE YOUR PROBLEM HERE — e.g., "Audit and optimize cloud architecture proposals", "Due diligence on M&A data rooms", "Automated grant/RFP compliance & enrichment"]
- **Raw Inputs:** [DESCRIBE INPUT FORMATS — e.g., PDFs, XML, JSON, Markdown, code repo]
- **Target Deliverable(s):** [DESCRIBE OUTPUTS — e.g., sanitized Markdown blocks, JSON report, compliance matrix]
- **Default Swarm Scale:** `--num-workers 20` (must scale cleanly up to `100+` via CLI/env), `--max-concurrency 15`, `--swarm-mode tournament` (or `sharded`)
- **External Discovery Needed (Phase 2)?** [YES/NO — if YES, what should the Deep Scout search and rank?]

### Engineering Rules You Must Follow
1. **Ask Clarifying Questions First (if needed):** Before coding, confirm the 4–6 scoring dimensions (summing to 100 pts), whether `--swarm-mode` should default to `sharded` or `tournament`, and the hard zero-hallucination constraints.
2. **Tenant-Ready LLM Client:** Support `LLM_BASE_URL` / `OPENAI_BASE_URL` and tenant API keys via `.env` (no hardcoded provider URLs). Keep `--budget-usd` optional (telemetry-only when set to `0`).
3. **Elastic Worker Pool (`src/core/swarm_runner.py`):** Implement parameterized `--num-workers` (20–100+) decoupled from `--max-concurrency` using `asyncio.Semaphore` / `ThreadPoolExecutor` with `429 Retry-After` exponential backoff + jitter. Never let workers mutate shared state directly (use Shard-then-Reduce or Tournament Top-K selection).
4. **Phase 0 First (Zero-Token Baseline):** Build the deterministic parser + validator (`--stage phase0`) before wiring any LLM calls.
5. **Closed-Loop Convergence (Phase 1):** Implement epoch-by-epoch execution (`--max-iterations`, `--top-k`), hybrid deterministic + LLM critic scoring, and automatic Tier 1 -> Tier 2 model escalation on stagnation.
6. **Human-in-the-Loop Gate:** Never invent missing facts. Emit `[CONFIRM WITH USER: ...]` tags, store them in `input/user_confirmations.json`, and serve a local HTML form (`confirmations.html`) on the dashboard server.
7. **Thread-Safe Telemetry & Aggregated Dashboard:** Implement `src/core/telemetry.py` with a single-writer lock/queue writing atomically to `output/telemetry_live.json`, using aggregated `worker_pools` metrics so the zero-dependency SPA (`127.0.0.1:8765`) stays fast even at 100+ workers.
8. **Cross-Platform CLI Wrapper & Verification:** Provide `run.sh`, `run.bat`, `AGENTS.md`, and `.env.example`, then run Phase 0 and `--stage simulate` with `--num-workers 20` to verify everything end-to-end.
```
