# RiveDesNuages

> Cloud endpoint reference archive — "Shore of Clouds"
> Copied from `~/Desktop/agenda/` on 2026-05-12.
> This is a self-contained mirror of the kanban profile docs and reflections.
> **Last updated: 2026-05-21** — Ollama Cloud removed (refund collected, churn campaign successful). All profiles migrated off ollama-cloud to nous/opencode-go/openrouter fallbacks.

## Current State

As of 2026-05-21:
- 🚫 **ring-2.6-1t (OpenRouter)** — expired
- 🚫 **qwen3.6-plus (Nous)** — expired
- 🚫 **Ollama Cloud** — refunded & deprecated (hostile personnel, churn campaign successful). All profiles migrated off.
- ✅ **deepseek-v4-flash (nous)** — primary for `planner`, `inspector`, and `free` profiles
- ✅ **deepseek-v4-flash (opencode-go)** — primary for `CC` profile, fallback for `planner`/`inspector`/`free`
- ✅ **stepfun/step-3.5-flash (nous)** — last-resort fallback for `free`
- 📍 Ground truth config lives at `~/.hermes/config.yaml`

## Contents

| File | Origin | Purpose |
|---|---|---|
| `notes.md` | `~/Desktop/agenda/notes.md` | Full endpoint catalog, 15 endpoints, 4 profiles, pipeline mapping |
| `companion.md` | `~/Desktop/agenda/companion.md` | Anecdotes, design rationale, credential chain, hard-won truths |
| `profile-worker.yaml` | `~/.hermes/profiles/worker/config.yaml` | Execution engine: `kimi-k2.6` on kimi-coding → nous fallback |
| `profile-planner.yaml` | `~/.hermes/profiles/planner/config.yaml` | Reasoning: `deepseek-v4-flash` on nous → opencode-go fallback |
| `profile-inspector.yaml` | `~/.hermes/profiles/inspector/config.yaml` | QA: `deepseek-v4-flash` on nous → opencode-go fallback |
| `profile-free.yaml` | `~/.hermes/profiles/free/config.yaml` | Zero-cost research: `deepseek-v4-flash` on nous → openrouter → opencode-go → stepfun/step-3.5-flash |
| `profile-cc.yaml` | `~/.hermes/profiles/cc/config.yaml` | CC profile — `deepseek-v4-flash` on opencode-go → openrouter/nous fallback |

## Notes
- These are **snapshot copies**, not symlinks. If the originals change, this folder will drift.
- **Ollama Cloud is gone** — refund collected, churn campaign successful. All profiles migrated to nous/opencode-go/openrouter. No more ollama-cloud references anywhere.