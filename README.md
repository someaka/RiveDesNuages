# RiveDesNuages

> Cloud endpoint reference archive — "Shore of Clouds"
> Copied from `~/Desktop/agenda/` on 2026-05-12.
> This is a self-contained mirror of the kanban profile docs and reflections.
> **Last updated: 2026-05-15** — ring-2.6-1t and qwen3.6-plus expired. DeepSeek V4 Flash on Nous is now the sole free universal fallback.

## Current State

As of 2026-05-15, the free model landscape shifted:
- 🚫 **ring-2.6-1t (OpenRouter)** — expired
- 🚫 **qwen3.6-plus (Nous)** — expired
- ✅ **DeepSeek V4 Flash (Nous)** — now the primary for `free` profile, model name `deepseek/deepseek-v4-flash`
- ✅ **stepfun/step-3.5-flash (Nous)** — new fallback for `free` profile, proven in auxiliary tasks
- 📍 Ground truth config lives at `~/.hermes/config.yaml` — single-model: deepseek v4 flash on Nous with OpenRouter fallback

## Contents

| File | Origin | Purpose |
|---|---|---|
| `notes.md` | `~/Desktop/agenda/notes.md` | Full endpoint catalog, 15 endpoints, 4 profiles, pipeline mapping |
| `companion.md` | `~/Desktop/agenda/companion.md` | Anecdotes, design rationale, credential chain, hard-won truths |
| `profile-worker.yaml` | `~/.hermes/profiles/worker/config.yaml` | K2.6 execution engine (Moonshots → Ollama fallback) |
| `profile-planner.yaml` | `~/.hermes/profiles/planner/config.yaml` | DS4Pro reasoning (Ollama → OpenCode fallback) |
| `profile-inspector.yaml` | `~/.hermes/profiles/inspector/config.yaml` | GLM5.1 QA (Ollama → Qwen3.6 → DeepSeek fallback) |
| `profile-free.yaml` | `~/.hermes/profiles/free/config.yaml` | Zero-cost research (ring-2.6-1t → Qwen3.6 → DeepSeek) |

## Notes
- These are **snapshot copies**, not symlinks. If the originals change, this folder will drift.