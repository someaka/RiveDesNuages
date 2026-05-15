# Endpoint Notes

> Compiled from conversation on 2026-05-11. Ongoing — updated as user rambles through.
> Last update: 2026-05-15 — ring-2.6-1t and qwen3.6-plus expired, DeepSeek V4 Flash on Nous promoted to universal fallback.
> Mirror copy — original lives at ~/Desktop/agenda/notes.md
> Ground truth config: ~/.hermes/config.yaml (single-model: deepseek/deepseek-v4-flash on Nous)

---

## KANBAN PIPELINE — OPERATIONAL FLOW

The full delivery pipeline, mapped to profiles. Each stage has a **gate** — criteria that must be met before handoff to the next stage.

```
 ┌──────────────┐    ┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────────┐
 │ INVESTIGATE  │───→│ SPECS    │───→│ PLAN    │───→│ VERIFY PLAN  │───→│ EXECUTE      │
 │              │    │          │    │         │    │              │    │              │
 │ free         │    │ planner  │    │ planner │    │ inspector    │    │ worker       │
 │ DS4Flash     │    │ DS4Pro   │    │ DS4Pro  │    │ GLM5.1       │    │ K2.6         │
 │ (zero cost)  │    │ Ollama   │    │         │    │ Ollama Cloud │    │ Moonshots    │
 └──────┬───────┘    └────┬─────┘    └────┬────┘    └──────┬───────┘    └──────┬───────┘
        │                  │               │                │                   │
        │                  │               │                │                   │
        │    GATE: problem │  GATE: task   │  GATE: plan   │  GATE: plan       │
        │    understood    │  decomposed   │  reviewed &   │  approved as      │
        │    & scoped      │  into subtasks│  validated    │  viable           │
        ▼                  ▼               ▼                ▼                   ▼
 ┌──────────────┐    ┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────────┐
 │ VERIFY       │←───│ FIX &    │←───│ INSPECT │←───│ VERIFY       │    │ NOTIFY USER  │
 │ EXECUTION    │    │ POLISH   │    │         │    │ EXECUTION    │◄───│ FOR AUDIT    │
 │              │    │          │    │ inspect │    │              │    │              │
 │ inspector    │    │ worker   │    │ GLM5.1  │    │ inspector    │    │ (final       │
 │ GLM5.1       │    │ K2.6     │    │ Ollama  │    │ GLM5.1       │    │  output)     │
 │ Cloud        │    │ Moonshots│    │ Cloud   │    │ Ollama Cloud │    │              │
 └──────┬───────┘    └────┬─────┘    └────┬────┘    └──────┬───────┘    └──────────────┘
        │                  │               │                │
        │                  │               │                │
        │                  │               │                │
        └──────────────────┴───────────────┴────────────────┘
                          LOOP BACK if issues found
```

### Stage Definitions

#### 1. INVESTIGATE — `free` profile (DeepSeek V4 Flash)
- **Goal:** Understand the problem space. Gather context, research, explore.
- **Profile:** `free` ← zero cost, lightweight, no heavy tools
- **Model:** `deepseek/deepseek-v4-flash` (Nous — direct). Fallback: `stepfun/step-3.5-flash` (Nous).
- **Toolsets:** web search, memory, session search, file read, vision
- **Output:** Problem statement, constraints, scope definition
- **Handoff gate:** Problem is clearly defined. Scope is bounded.

#### 2. SPECS — `planner` profile (DS4Pro)
- **Goal:** Translate investigation into technical specifications. Define what needs building.
- **Profile:** `planner` ← DS4Pro reasoning on Ollama Cloud (with OpenCode Go fallback)
- **Toolsets:** web, memory, session search, code_execution (validation), file (read-only), vision
- **Output:** Detailed specs document — inputs, outputs, edge cases, interfaces
- **Handoff gate:** Specs are complete, testable, and reviewed.

#### 3. PLAN — `planner` profile (DS4Pro)
- **Goal:** Break the work into executable tasks. Decompose into a task graph.
- **Profile:** `planner` ← same, deeper reasoning pass
- **Toolsets:** Same as specs + session_search for prior context
- **Output:** Task decomposition — ordered steps, dependencies, estimated complexity
- **Handoff gate:** Each task is atomic enough for K2.6 to execute independently.

#### 4. VERIFY PLAN — `inspector` profile (GLM5.1)
- **Goal:** QA the plan before execution. Catch gaps, contradictions, missing edge cases.
- **Profile:** `inspector` ← GLM5.1 on Ollama Cloud (with Qwen3.6 Plus fallback)
- **Toolsets:** file (read specs + plan), memory (historical context), code_execution (feasibility checks)
- **Output:** Plan review with approved/blocker flags
- **Handoff gate:** Plan passes inspection. All blockers resolved.

#### 5. EXECUTE — `worker` profile (K2.6)
- **Goal:** Build/implement. Rip through the task decomposition.
- **Profile:** `worker` ← K2.6 on Moonshots (with Ollama Cloud fallback)
- **Toolsets:** Full system — terminal, file, code_execution, web, memory, vision, image_gen, tts, cronjob
- **Output:** Implementation artifacts — code, configs, files
- **Handoff gate:** All tasks marked complete. Execution log available.

#### 6. VERIFY EXECUTION — `inspector` profile (GLM5.1)
- **Goal:** QA the output. Test it. Find bugs, gaps, regressions.
- **Profile:** `inspector` ← same
- **Toolsets:** file (read output), code_execution (run tests/verify), memory (compare against specs)
- **Output:** Execution review — pass/fail per task, bug list
- **Handoff gate:** Either ✅ all pass, or 🔁 loop back to FIX & POLISH

#### 7. FIX & POLISH — `worker` profile (K2.6)
- **Goal:** Address inspector findings. Fix bugs, fill gaps, clean up.
- **Profile:** `worker` ← same
- **Toolsets:** Full system access
- **Output:** Patched artifacts
- **Handoff gate:** All fixes applied. Re-submit for inspection.

#### 8. INSPECTION — `inspector` profile (GLM5.1)
- **Goal:** Final deep review. End-to-end audit of deliverables.
- **Profile:** `inspector` ← same
- **Toolsets:** Full read + verify toolkit
- **Output:** Final quality report

#### 9. VERIFICATION — `inspector` profile (GLM5.1)
- **Goal:** Confirm everything meets the original specs. Trace back to step 2.
- **Profile:** `inspector` ← same
- **Output:** Sign-off or loop back to fix

#### 10. FIX — `worker` profile (K2.6)
- **Goal:** Last-mile fixes before user delivery.
- **Profile:** `worker` ← same
- **Output:** Final build

#### 11. NOTIFY USER FOR AUDIT
- **Goal:** Deliver the output. User reviews, audits, accepts or requests changes.
- **No profile needed** — this is the human handoff point.
- **Output:** Summary + deliverables + any open questions

---

## PIPELINE × PROFILE MATRIX

| Stage | Profile | Model | Provider | Cost Path | Status |
|---|---|---|---|---|---|---|
| 1. Investigate | `free` | DeepSeek V4 Flash | Nous | $0 (may be temp) | ✅ Active |
| 2. Specs | `planner` | DS4Pro | Ollama Cloud | $15/yr | ✅ Active |
| 3. Plan | `planner` | DS4Pro | Ollama Cloud | $15/yr | ✅ Active |
| 4. Verify Plan | `inspector` | GLM5.1:cloud | Ollama Cloud | $15/yr | ✅ Active |
| 5. Execute | `worker` | K2.6 | Moonshots | $40/mo | ✅ Active |
| 6. Verify Execution | `inspector` | GLM5.1:cloud | Ollama Cloud | $15/yr | ✅ Active |
| 7. Fix & Polish | `worker` | K2.6 | Moonshots | $40/mo | ✅ Active |
| 8. Inspection | `inspector` | GLM5.1:cloud | Ollama Cloud | $15/yr | ✅ Active |
| 9. Verification | `inspector` | GLM5.1:cloud | Ollama Cloud | $15/yr | ✅ Active |
| 10. Fix | `worker` | K2.6 | Moonshots | $40/mo | ✅ Active |
| 11. Notify | — | — | — | — | Human handoff |

**Cost per full cycle (current):**
- Ollama Cloud: ~$1.25/cycle (light usage on unlimited $15/yr plan)
- Moonshots: ~$1.33/cycle (light usage on $40/mo plan)
- Free tier (investigation): **$0** — DeepSeek V4 Flash on Nous (free for now)
- Fallback (free tier): **$0** — stepfun/step-3.5-flash on Nous (proven in aux tasks)
- Total: essentially pennies per cycle — investigation stage has two free fallback layers

---

## OPERATIONAL SCHEDULING NOTES

### Ollama Cloud — Peak Hours (Europe)
- **Heavy-duty window:** 13:00–18:00 UTC+2 (Europe's peak usage)
- **Schedule pipeline stages 2–4, 6, 8–9 OUTSIDE this window**
- **Applies to:** inspector (GLM5.1), planner (DS4Pro), worker fallback path (K2.6 on Ollama)

### MIMO — Off-Peak Sweet Spot
- **Optimal window:** 16:00–02:00 UTC+2
- **Not wired into pipeline** — break-glass only

### K2.6 — Off-Peak (Moonshots Direct)
- **Likely window:** 16:00–02:00 UTC+2 (shares Moonshots infra with MIMO)
- **Schedule stages 5, 7, 10 in this window when possible**

### Nous / OpenCode Go
- **No known peak constraints** — routing is global / CDN-backed
- **Note:** Nous free-tier offers are temporary; OpenCode Go $5 allocation has an "infinity" window that will expire

### Recommended Pipeline Schedule (UTC+2)
| Time | Activity |
|---|---|
| 02:00–12:00 | Pipeline execution — stages 1–6 while US/Asia infra is fresh |
| 12:00–13:00 | Buffer / handoff window |
| 13:00–18:00 | **Ollama Cloud peak — AVOID heavy pipeline stages** |
| 16:00–02:00 | K2.6/MIMO off-peak — best for stages 5, 7, 10 on Moonshots |
| 18:00–02:00 | Pipeline execution resumes — stages 6–11 |

---

## FREE ENDPOINTS

### ~~1. ring-2.6-1t (OpenRouter) — EXPIRED~~
- **Provider:** OpenRouter (inclusionai/ring-2.6-1t)
- **Cost:** Free — temporary offer
- **Status:** ❌ **Expired** — offer ended
- **Performance:** Was outstanding TPS. Model card claimed it could trade blows with Opus 4.7. Fastest TPS on OpenRouter at the time.
- **Role in pipeline:** Was Stage 1 (Investigate) — zero-cost research leg. Now gone.
- **Removed from:** All profiles, all fallback chains.

### ~~2. qwen3.6-plus (Nous) — EXPIRED~~
- **Provider:** Nous Portal
- **Cost:** Free — temporary offer
- **Status:** ❌ **Expired** — Nous portal freebie ended
- **Role in kanban stack:** Was universal auxiliary fallback for all four profiles. Now gone.
- **Removed from:** All profiles, all fallback chains. DeepSeek V4 Flash replaces it.

### 3. deepseek-v4 flash (Nous) ← CURRENT UNIVERSAL FALLBACK
- **Provider:** Nous Portal (direct — https://inference-api.nousresearch.com/v1)
- **Cost:** Free (currently)
- **Status:** ⚠️ Active now — may be temporary, only time will tell
- **Role in kanban stack:** Universal fallback for all profiles. Also the **current default model** in `config.yaml`.
- **Notes:** Replaces both ring-2.6-1t and qwen3.6-plus in the stack. Available directly on Nous inference API, model name `deepseek/deepseek-v4-flash`. OpenRouter fallback uses `deepseek/deepseek-v4-flash:free`.

### 4. nemotron3 (OpenRouter)
- **Provider:** OpenRouter | **Cost:** Free (always) | **Status:** Legacy / permanent

### 5. openrouter:free (OpenRouter)
- **Provider:** OpenRouter | **Cost:** Free (always) | **Status:** Legacy routing wrapper
- **Note:** Separate from nemotron3 — confirmed distinct.

### 6. arcee (Nous)
- **Provider:** Nous Portal | **Cost:** Free | **Status:** Available
- **Quality:** Last pick when a model is required and no other free option is available.

---

## RADAR — TESTED & WORKING, NOT DEPLOY-READY

### 7. NVIDIA NIM
- **Status:** ⛔ Europe geo-restricted | **TPS:** Too low

### 8. openrouter:owl-alpha (OpenRouter)
- **Status:** Tested, functional | **TPS:** Too low

---

## LOCAL / HOSTED MODELS (Own Hardware) 🔧

### 9. qwen3.6 27B (Local) — 🔬 Benchmarking
### 10. qwen3.6 35B A3B (Local) — 🔬 Benchmarking
- Self-hosted, $0 inference cost. Promising results, not yet promoted.
- If either proves reliable, they absorb free-tier roles and eliminate external dependency.

---

## PAID ENDPOINTS

### 11. Ollama Cloud ⭐ CORE PLATFORM
- **Cost:** $15/year Pro plan — unlimited quotas
- **Models hosted:** glm5.1:cloud, k2.6, ds4pro
- **Role:** Pipeline stages 2–4 (planner + inspector), worker fallback
- **⚠️ Peak hours:** 13:00–18:00 UTC+2

### 12. opencode:go 🔥 TIER-1 FAILOVER
- **Cost:** $5 (promo) + "infinity" temporary bonus
- **TPS:** Fastest of all listed endpoints
- **Allocation:** ~1,200 MIMO/Kimi / ~3,000+ DS4Pro / ~31,000 DeepSeek V4 Flash
- **Role:** Planner failover + universal last-resort endpoint

### 13. Moonshots Direct — K2.6
- **Cost:** $40/month Allegro plan — autorenewing
- **Role:** Pipeline stages 5, 7, 10 (execution legs)
- **Fallback:** Ollama Cloud K2.6

### 14. MIMO Endpoint
- **Cost:** $100/mo standard / $77 new-customer / potentially $70/mo (Discord pending)
- **Role:** Break-glass emergency only — manual use, not in pipeline routing
- **Off-peak:** 16:00–02:00 UTC+2

### 15. ZenMux
- **Cost:** ~$20/month | **Status:** 🆕 Evaluating | **TPS:** Impressive
- **Notes:** Great model roster, sketchy Vibe-coded site. Sustained TPS testing needed.

---

## SUMMARY MATRIX

| # | Endpoint | Provider | Cost | Avail. | TPS | Category | Verdict |
|---|---|---|---|---|---|---|---|---|
|| ~~1~~ | ~~ring-2.6-1t~~ | ~~OpenRouter~~ | ~~Free~~ | ❌ **Expired** | ~~Outstanding~~ | ~~Free~~ | ❌ Dead |
|| ~~2~~ | ~~qwen3.6-plus~~ | ~~Nous~~ | ~~Free~~ | ❌ **Expired** | — | ~~Free~~ | ❌ Dead |
| 1 | **deepseek-v4 flash** | **Nous** | **Free (now)** | ⚠️ Temporary | — | Free | ✅ **Default model + universal fallback** |
| 2 | nemotron3 | OpenRouter | Free (always) | ✅ Stable | — | Free | ⚠️ Low quality fallback |
| 3 | openrouter:free | OpenRouter | Free (always) | ✅ Stable | — | Free | ⚠️ Lowest quality router |
| 4 | arcee | Nous | Free | ✅ Available | — | Free | 🟡 Last pick |
| 5 | NVIDIA NIM | NVIDIA | Usage-based | ⛔ Blocked | ⚠️ Low | Radar | 🔒 Geo-restricted |
| 6 | owl-alpha | OpenRouter | Unknown | 📋 Tested | ⚠️ Low | Radar | 🔒 Monitoring |
| 7 | qwen3.6 27B | **Local** | **$0** | ✅ Always | TBD | **Local** | 🔬 Benchmarking |
| 8 | qwen3.6 35B A3B | **Local** | **$0** | ✅ Always | TBD | **Local** | 🔬 Benchmarking |
| 9 | Ollama Cloud | Ollama | **$15/year** | ✅ Active | ⚠️ Lowest | Paid | ✅ Core platform |
| 10 | opencode:go | OpenCode | **$5** + infinity | 🔥 Active | ⭐ Fastest | Paid | ✅ Planner failover |
| 11 | K2.6 (Moonshots) | Moonshots (Allegro) | **$40/month** | ✅ Active | ⭐ High | Paid | ✅ Worker primary |
| 12 | MIMO (Moonshots) | Moonshots direct | **$100/mo ($77 new)** | 🔒 Personal | — | Paid | 🔥 Break-glass |
| 13 | ZenMux | ZenMux | ~$20/month | 🆕 Eval. | ⭐ Impressive | Paid | ⚠️ Great roster, sketchy site |

---

## KANBAN PROFILES — FOUR PROFILES, UNIVERSAL FALLBACK CHAIN

### Profile → Pipeline Mapping
```
Stage 1  (Investigate)  →  free        →  deepseek/deepseek-v4-flash  (Nous)
Stage 2  (Specs)        →  planner     →  ds4pro           (Ollama Cloud)
Stage 3  (Plan)         →  planner     →  ds4pro           (Ollama Cloud)
Stage 4  (Verify Plan)  →  inspector   →  glm5.1:cloud     (Ollama Cloud)
Stage 5  (Execute)      →  worker      →  k2.6             (Moonshots)
Stage 6  (Verify Exec)  →  inspector   →  glm5.1:cloud     (Ollama Cloud)
Stage 7  (Fix & Polish) →  worker      →  k2.6             (Moonshots)
Stage 8  (Inspection)   →  inspector   →  glm5.1:cloud     (Ollama Cloud)
Stage 9  (Verification) →  inspector   →  glm5.1:cloud     (Ollama Cloud)
Stage 10 (Final Fix)    →  worker      →  k2.6             (Moonshots)
Stage 11 (Auditor)      →  YOU         →  human review
```

### 🆓 `free` — Zero-Cost Research Profile (Resurrected)
- **Model:** deepseek/deepseek-v4-flash
- **Primary:** Nous (direct — https://inference-api.nousresearch.com/v1)
- **Fallback:** stepfun/step-3.5-flash (Nous — same base URL, proven in auxiliary tasks)
- **Character:** Helpful, reasoning shown, lighter toolset (no code execution, no system tools)
- **Max turns:** 120
- **Pipeline role:** Stage 1 — Investigate
- **Status:** ✅ **Active** — replaced expired ring-2.6-1t and qwen3.6-plus with deepseek flash + stepfun fallback

### 🧠 `planner` — DS4Pro Reasoning
- **Model:** ds4pro
- **Primary:** Ollama Cloud
- **Fallback 1:** DeepSeek V4 Flash (Nous — direct)
- **Fallback 2:** OpenCode Go DS4Pro
- **Character:** Technical, full reasoning visible
- **Max turns:** 200
- **Pipeline role:** Stages 2–3 (Specs + Plan)

### 🔧 `worker` — K2.6 Execution Engine
- **Model:** k2.6
- **Primary:** Moonshots
- **Fallback 1:** Ollama Cloud K2.6
- **Fallback 2:** DeepSeek V4 Flash (Nous — direct)
- **Character:** Concise, no reasoning fluff, tool-use enforced
- **Max turns:** 60
- **Pipeline role:** Stages 5, 7, 10 (Execute + Fix + Polish + Final Fix)

### 🔍 `inspector` — GLM5.1 QA Review
- **Model:** glm5.1:cloud
- **Primary:** Ollama Cloud
- **Fallback 1:** DeepSeek V4 Flash (Nous — direct)
- **Character:** Helpful, thorough, shows reasoning for audit trails
- **Max turns:** 300
- **Pipeline role:** Stages 4, 6, 8, 9 (Verify Plan + Verify Execution + Inspection + Verification)

---

## ⚠️ Remaining Confirmations
1. ~~**DeepSeek V4 Flash model name** — `deepseek-v4-flash` in OpenCode Go?~~ ✅ **Resolved** — It's `deepseek/deepseek-v4-flash` on Nous, and `deepseek/deepseek-v4-flash:free` on OpenRouter fallback.
2. ~~**Nous Qwen3.6 Plus model name** — `qwen3.6-plus` confirmed?~~ ❌ **Moot** — qwen3.6-plus expired. Not needed anymore.
3. ~~**ring-2.6-1t** — will it last?~~ ❌ **Expired** — answer came.
4. **ZenMux evaluation** — pending TPS / catalog mapping
5. **Phase 1 specialist roster** — central planning artifact
6. **Moonshots Discord** — MIMO $70/month answer
7. **Rate limits / expirations** on remaining free offers (DeepSeek V4 Flash on Nous)
8. **Local Qwen benchmark results** — awaiting promotion decision
9. ~~**What replaces the `free` profile?**~~ ✅ **Resolved** — DeepSeek V4 Flash (primary) + stepfun/step-3.5-flash (fallback), both on Nous.