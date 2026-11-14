# Elastic Closed-Loop Swarm Recipe (Instructions Only)

Domain-agnostic, harness-agnostic, and enterprise-tenant-ready instruction pack for scaffolding **20–100+ parallel AI workers** with **deterministic reward control**, **zero-hallucination HITL gates**, and a **live Mission Control dashboard**.

## Files
- `SWARM_BLUEPRINT.md` — Full architecture specification (`--num-workers`, `--max-concurrency`, `--swarm-mode sharded|tournament`, `--top-k`, `telemetry_live.json` contract, and enterprise tenant routing).
- `BOOTSTRAP_PROMPT.md` — Copy-paste master prompt to instruct any AI coding harness (Claude Code, Codex, Cursor, Gemini CLI, OpenCode, Antigravity) to scaffold a new project from scratch.
- `AGENTS_TEMPLATE.md` — Drop-in `AGENTS.md` template for newly scaffolded projects.
