# RiveDesNuages

> Cloud endpoint reference archive — "Shore of Clouds"
> Copied from `~/Desktop/agenda/` on 2026-05-12.
> This is a self-contained mirror of the kanban profile docs and reflections.
> **Last updated: 2026-05-19** — Ollama Cloud being phased out (hostile personnel, FA → FO escalation active on all social networks). Planner model fixed (ds4pro → deepseek-v4-pro). New CC profile added. Free fallback chain: OpenRouter DS4Flash → OpenCode Go → Nous stepfun.

## Current State

As of 2026-05-19:
- 🚫 **ring-2.6-1t (OpenRouter)** — expired
- 🚫 **qwen3.6-plus (Nous)** — expired
- ⚠️  **Ollama Cloud** — being phased out (hostile moderator behavior, FA → FO escalation live). **MUST USE IN PRIORITY WHILE ANY QUOTA REMAINS.** Refund pending; if silence persists → escalation continues until resolution or cut-off. Once refunded/cut-off: immediately pivot planner/inspector/CC to fallback providers.
- ✅ **deepseek-v4-pro (ollama-cloud)** — primary for `planner` and `CC` profiles
- ✅ **deepseek-v4-flash (nous)** — primary for `free` profile
- ✅ **deepseek-v4-flash (openrouter, `deepseek/deepseek-v4-flash:free`)** — first fallback for `free`
- ✅ **deepseek-v4-flash (opencode-go)** — second fallback for `free`
- ✅ **stepfun/step-3.5-flash (nous)** — last-resort fallback for `free`
- 📍 Ground truth config lives at `~/.hermes/config.yaml`

## Contents

| File | Origin | Purpose |
|---|---|---|
| `notes.md` | `~/Desktop/agenda/notes.md` | Full endpoint catalog, 15 endpoints, 4 profiles, pipeline mapping |
| `companion.md` | `~/Desktop/agenda/companion.md` | Anecdotes, design rationale, credential chain, hard-won truths |
| `profile-worker.yaml` | `~/.hermes/profiles/worker/config.yaml` | Execution engine: `kimi-k2.6` on kimi-coding → ollama-cloud fallback |
| `profile-planner.yaml` | `~/.hermes/profiles/planner/config.yaml` | Reasoning: `deepseek-v4-pro` on ollama-cloud → opencode-go fallback |
| `profile-inspector.yaml` | `~/.hermes/profiles/inspector/config.yaml` | QA: `glm-5.1:cloud` on ollama-cloud → deepseek-v4-flash fallback |
| `profile-free.yaml` | `~/.hermes/profiles/free/config.yaml` | Zero-cost research: `deepseek-v4-flash` on nous → openrouter → opencode-go → stepfun/step-3.5-flash |
| `profile-cc.yaml` | `~/.hermes/config.yaml` | 🆕 CC profile — `deepseek-v4-pro` on ollama-cloud (burn quotas), full toolset, default companion |

## Notes
- These are **snapshot copies**, not symlinks. If the originals change, this folder will drift.
- **Ollama Cloud profiles (`planner`, `inspector`, `CC`) are on borrowed time** — deplete quotas before service cutoff or refund. Ollama gets PRIORITY on all profiles that use it (planner, inspector, CC, worker fallback). Fallback providers are wired but NOT hit until Ollama is truly dry. The moment quotas hit zero or service cuts off: pivot all profiles to their respective fallback chains immediately.