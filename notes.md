# Endpoint Notes

> Compiled from conversation on 2026-05-11. Ongoing — updated as user rambles through.
> Last update: 2026-05-19 — Ollama Cloud phase-out (FA→FO), CC profile added, opencode-go in free fallback chain, ds4pro→deepseek-v4-pro.
> Mirror copy — original lives at ~/Desktop/agenda/notes.md
> Ground truth config: ~/.hermes/config.yaml (single-model: deepseek-v4-pro on ollama-cloud)

---

## KANBAN PIPELINE — OPERATIONAL FLOW

The full delivery pipeline, mapped to profiles. Each stage has a **gate** — criteria that must be met before handoff to the next stage.

```
 ┌──────────────┐    ┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────────┐
 │ INVESTIGATE  │───→│ SPECS    │───→│ PLAN    │───→│ VERIFY PLAN  │───→│ EXECUTE      │
 │              │    │          │    │         │    │              │    │              │
 │ free         │    │ planner  │    │ planner │    │ inspector    │    │ worker       │
 │ deepseek-    │    │ deepseek-│    │         │    │ glm-5.1:cloud│    │ kimi-k2.6    │
 │ v4-flash     │    │ v4-pro   │    │         │    │ ollama-cloud │    │ kimi-coding  │
 │ (nous)       │    │ ollama   │    │         │    │ (⚠️ phase-out)│   │              │
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
 │ inspector    │    │ worker   │    │ inspect │    │ inspector    │    │              │
 │ glm-5.1:cloud│    │ kimi-k2.6│    │         │    │ glm-5.1:cloud│    │ (final       │
 │ ollama-cloud │    │ kimi-    │    │ ollama- │    │ ollama-cloud │    │  output)     │
 │ (⚠️ phase-out)│   │ coding   │    │ cloud   │    │ (⚠️ phase-out)│   │              │
        │                  │               │                │
        │                  │               │                │
        │                  │               │                │
        └──────────────────┴───────────────┴────────────────┘
                          LOOP BACK if issues found
```

### Stage Definitions

#### 1. INVESTIGATE — `free` profile (deepseek-v4-flash)
- **Goal:** Understand the problem space. Gather context, research, explore.
- **Profile:** `free` ← zero cost, lightweight, no heavy tools
- **Model:** `deepseek/deepseek-v4-flash` (nous). Fallbacks: `deepseek/deepseek-v4-flash:free` (openrouter) → `deepseek-v4-flash` (opencode-go) → `stepfun/step-3.5-flash` (nous).
- **Toolsets:** web search, memory, session search, file read, vision
- **Output:** Problem statement, constraints, scope definition
- **Handoff gate:** Problem is clearly defined. Scope is bounded.

#### 2. SPECS — `planner` profile (deepseek-v4-pro)
- **Goal:** Translate investigation into technical specifications. Define what needs building.
- **Profile:** `planner` ← deepseek-v4-pro reasoning on ollama-cloud (⚠️ phase-out, deplete quotas first), opencode-go fallback
- **Toolsets:** web, memory, session search, code_execution (validation), file (read-only), vision
- **Output:** Detailed specs document — inputs, outputs, edge cases, interfaces
- **Handoff gate:** Specs are complete, testable, and reviewed.

#### 3. PLAN — `planner` profile (deepseek-v4-pro)
- **Goal:** Break the work into executable tasks. Decompose into a task graph.
- **Profile:** `planner` ← same, deeper reasoning pass
- **Toolsets:** Same as specs + session_search for prior context
- **Output:** Task decomposition — ordered steps, dependencies, estimated complexity
- **Handoff gate:** Each task is atomic enough for kimi-k2.6 to execute independently.

#### 4. VERIFY PLAN — `inspector` profile (glm-5.1:cloud)
- **Goal:** QA the plan before execution. Catch gaps, contradictions, missing edge cases.
- **Profile:** `inspector` ← glm-5.1:cloud on ollama-cloud (⚠️ phase-out), deepseek-v4-flash fallback
- **Toolsets:** file (read specs + plan), memory (historical context), code_execution (feasibility checks)
- **Output:** Plan review with approved/blocker flags
- **Handoff gate:** Plan passes inspection. All blockers resolved.

#### 5. EXECUTE — `worker` profile (kimi-k2.6)
- **Goal:** Build/implement. Rip through the task decomposition.
- **Profile:** `worker` ← kimi-k2.6 on kimi-coding, ollama-cloud fallback
- **Toolsets:** Full system — terminal, file, code_execution, web, memory, vision, image_gen, tts, cronjob
- **Output:** Implementation artifacts — code, configs, files
- **Handoff gate:** All tasks marked complete. Execution log available.

#### 6. VERIFY EXECUTION — `inspector` profile (glm-5.1:cloud)
- **Goal:** QA the output. Test it. Find bugs, gaps, regressions.
- **Profile:** `inspector` ← same
- **Toolsets:** file (read output), code_execution (run tests/verify), memory (compare against specs)
- **Output:** Execution review — pass/fail per task, bug list
- **Handoff gate:** Either ✅ all pass, or 🔁 loop back to FIX & POLISH

#### 7. FIX & POLISH — `worker` profile (kimi-k2.6)
- **Goal:** Address inspector findings. Fix bugs, fill gaps, clean up.
- **Profile:** `worker` ← same
- **Toolsets:** Full system access
- **Output:** Patched artifacts
- **Handoff gate:** All fixes applied. Re-submit for inspection.

#### 8. INSPECTION — `inspector` profile (glm-5.1:cloud)
- **Goal:** Final deep review. End-to-end audit of deliverables.
- **Profile:** `inspector` ← same
- **Toolsets:** Full read + verify toolkit
- **Output:** Final quality report

#### 9. VERIFICATION — `inspector` profile (glm-5.1:cloud)
- **Goal:** Confirm everything meets the original specs. Trace back to step 2.
- **Profile:** `inspector` ← same
- **Output:** Sign-off or loop back to fix

#### 10. FIX — `worker` profile (kimi-k2.6)
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
|| 1. Investigate | `free` | `deepseek-v4-flash` | nous | $0 (temporary) | ✅ Active |
|| 2. Specs | `planner` | `deepseek-v4-pro` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 3. Plan | `planner` | `deepseek-v4-pro` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 4. Verify Plan | `inspector` | `glm-5.1:cloud` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 5. Execute | `worker` | `kimi-k2.6` | kimi-coding | $40/mo | ✅ Active |
|| 6. Verify Execution | `inspector` | `glm-5.1:cloud` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 7. Fix & Polish | `worker` | `kimi-k2.6` | kimi-coding | $40/mo | ✅ Active |
|| 8. Inspection | `inspector` | `glm-5.1:cloud` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 9. Verification | `inspector` | `glm-5.1:cloud` | ollama-cloud | $15/yr (⚠️ phase-out) | ✅ Active |
|| 10. Fix | `worker` | `kimi-k2.6` | kimi-coding | $40/mo | ✅ Active |
| 11. Notify | — | — | — | — | Human handoff |

**Cost per full cycle (current):**
- ollama-cloud: ~$1.25/cycle (light usage on $15/yr plan — ⚠️ phase-out, burn quotas)
- kimi-coding: ~$1.33/cycle (light usage on $40/mo plan)
- Free tier (investigation): **$0** — deepseek-v4-flash on nous (may be temporary) → openrouter → opencode-go → stepfun/step-3.5-flash
- Total: essentially pennies per cycle — investigation stage has four fallback layers

---

## OPERATIONAL SCHEDULING NOTES

### Ollama Cloud — Peak Hours (Europe) + ⚠️ PHASE-OUT
- **Status:** Being phased out (hostile moderator behavior, FA→FO escalation on all social networks). Refund pending.
- **Rule:** MUST USE IN PRIORITY while any quota remains. Fallback providers are wired but NOT hit until quotas are truly dry.
- **Heavy-duty window:** 13:00–18:00 UTC+2 (Europe's peak usage)
- **Schedule pipeline stages 2–4, 6, 8–9 OUTSIDE this window**
- **Applies to:** inspector (glm-5.1:cloud), planner (deepseek-v4-pro), worker fallback path

### MIMO — Off-Peak Sweet Spot
- **Optimal window:** 16:00–02:00 UTC+2
- **Not wired into pipeline** — break-glass only

### K2.6 — Off-Peak (Moonshots Direct)
- **Likely window:** 16:00–02:00 UTC+2 (shares Moonshots infra with MIMO)
- **Schedule stages 5, 7, 10 in this window when possible**

### Nous / OpenCode Go / OpenRouter
- **No known peak constraints** — routing is global / CDN-backed
- **Note:** nous free-tier offers are temporary; opencode-go $5 allocation has an "infinity" window that will expire

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

### 3. deepseek-v4-flash (nous) ← PRIMARY FREE MODEL
- **Provider:** nous (direct — https://inference-api.nousresearch.com/v1)
- **Cost:** Free (currently, may be temporary)
- **Status:** ✅ Active — `deepseek/deepseek-v4-flash`
- **Role:** Primary for `free` profile. First choice for zero-cost investigation.
- **Fallbacks:** openrouter (`deepseek/deepseek-v4-flash:free`) → opencode-go (`deepseek-v4-flash`) → stepfun/step-3.5-flash (nous)
- **Notes:** Replaces both ring-2.6-1t and qwen3.6-plus in the stack. Also served as universal fallback briefly before CC profile was introduced.

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

### 11. ollama-cloud ⭐ CORE PLATFORM — ⚠️ PHASE-OUT
- **Cost:** $15/year Pro plan — unlimited quotas
- **Models hosted:** glm-5.1:cloud, kimi-k2.6, deepseek-v4-pro
- **Role:** Pipeline stages 2–4 (planner + inspector), worker fallback
- **⚠️ Phase-out:** Hostile moderator behavior, FA→FO escalation on all social networks. Refund pending. MUST USE IN PRIORITY while any quota remains. Fallback providers wired but NOT hit until quotas dry.
- **⚠️ Peak hours:** 13:00–18:00 UTC+2

### 12. opencode-go 🔥 TIER-1 FAILOVER
- **Cost:** $5 (promo) + "infinity" temporary bonus
- **TPS:** Fastest of all listed endpoints
- **Allocation:** ~1,200 MIMO/Kimi / ~3,000+ deepseek-v4-pro / ~31,000 deepseek-v4-flash
- **Role:** Planner failover + free profile fallback (between openrouter and stepfun)

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
| 1 | **deepseek-v4-flash** | **nous** | **Free (now)** | ⚠️ Temporary | — | Free | ✅ **free profile primary** |
| 2 | nemotron3 | OpenRouter | Free (always) | ✅ Stable | — | Free | ⚠️ Low quality fallback |
| 3 | openrouter:free | OpenRouter | Free (always) | ✅ Stable | — | Free | ⚠️ Lowest quality router |
| 4 | arcee | Nous | Free | ✅ Available | — | Free | 🟡 Last pick |
| 5 | NVIDIA NIM | NVIDIA | Usage-based | ⛔ Blocked | ⚠️ Low | Radar | 🔒 Geo-restricted |
| 6 | owl-alpha | OpenRouter | Unknown | 📋 Tested | ⚠️ Low | Radar | 🔒 Monitoring |
| 7 | qwen3.6 27B | **Local** | **$0** | ✅ Always | TBD | **Local** | 🔬 Benchmarking |
| 8 | qwen3.6 35B A3B | **Local** | **$0** | ✅ Always | TBD | **Local** | 🔬 Benchmarking |
| 9 | ollama-cloud | Ollama | **$15/year** | ✅ Active | ⚠️ Lowest | Paid | ⚠️ Core platform — phase-out |
| 10 | opencode-go | OpenCode | **$5** + infinity | 🔥 Active | ⭐ Fastest | Paid | ✅ Planner failover + free fallback |
| 11 | K2.6 (Moonshots) | Moonshots (Allegro) | **$40/month** | ✅ Active | ⭐ High | Paid | ✅ Worker primary |
| 12 | MIMO (Moonshots) | Moonshots direct | **$100/mo ($77 new)** | 🔒 Personal | — | Paid | 🔥 Break-glass |
| 13 | ZenMux | ZenMux | ~$20/month | 🆕 Eval. | ⭐ Impressive | Paid | ⚠️ Great roster, sketchy site |

---

## KANBAN PROFILES — FOUR PROFILES, UNIVERSAL FALLBACK CHAIN

### Profile → Pipeline Mapping
```
Stage 1  (Investigate)  →  free        →  deepseek-v4-flash  (nous)
Stage 2  (Specs)        →  planner     →  deepseek-v4-pro    (ollama-cloud, ⚠️ phase-out)
Stage 3  (Plan)         →  planner     →  deepseek-v4-pro    (ollama-cloud, ⚠️ phase-out)
Stage 4  (Verify Plan)  →  inspector   →  glm-5.1:cloud      (ollama-cloud, ⚠️ phase-out)
Stage 5  (Execute)      →  worker      →  kimi-k2.6          (kimi-coding)
Stage 6  (Verify Exec)  →  inspector   →  glm-5.1:cloud      (ollama-cloud, ⚠️ phase-out)
Stage 7  (Fix & Polish) →  worker      →  kimi-k2.6          (kimi-coding)
Stage 8  (Inspection)   →  inspector   →  glm-5.1:cloud      (ollama-cloud, ⚠️ phase-out)
Stage 9  (Verification) →  inspector   →  glm-5.1:cloud      (ollama-cloud, ⚠️ phase-out)
Stage 10 (Final Fix)    →  worker      →  kimi-k2.6          (kimi-coding)
Stage 11 (Auditor)      →  YOU         →  human review
```

### 🆓 `free` — Zero-Cost Research Profile
- **Model:** `deepseek-v4-flash`
- **Primary:** nous (`deepseek/deepseek-v4-flash`)
- **Fallbacks:** openrouter (`deepseek/deepseek-v4-flash:free`) → opencode-go (`deepseek-v4-flash`) → stepfun/step-3.5-flash (nous)
- **Character:** Helpful, reasoning shown, lighter toolset (no code execution, no system tools)
- **Max turns:** 120
- **Pipeline role:** Stage 1 — Investigate
- **Status:** ✅ **Active** — four-layer fallback chain for zero-cost investigation

### 🧠 `planner` — DeepSeek V4 Pro Reasoning
- **Model:** `deepseek-v4-pro`
- **Primary:** ollama-cloud (⚠️ phase-out — deplete quotas first)
- **Fallback 1:** deepseek-v4-flash (nous)
- **Fallback 2:** deepseek-v4-pro (opencode-go)
- **Character:** Technical, full reasoning visible
- **Max turns:** 200
- **Pipeline role:** Stages 2–3 (Specs + Plan)

### 🔧 `worker` — Kimi K2.6 Execution Engine
- **Model:** `kimi-k2.6`
- **Primary:** kimi-coding
- **Fallback 1:** kimi-k2.6 (ollama-cloud, ⚠️ phase-out)
- **Fallback 2:** deepseek-v4-flash (nous)
- **Character:** Concise, no reasoning fluff, tool-use enforced
- **Max turns:** 60
- **Pipeline role:** Stages 5, 7, 10 (Execute + Fix + Polish + Final Fix)

### 🔍 `inspector` — GLM5.1 Cloud QA Review
- **Model:** `glm-5.1:cloud`
- **Primary:** ollama-cloud (⚠️ phase-out — deplete quotas first)
- **Fallback 1:** deepseek-v4-flash (nous)
- **Character:** Helpful, thorough, shows reasoning for audit trails
- **Max turns:** 300
- **Pipeline role:** Stages 4, 6, 8, 9 (Verify Plan + Verify Execution + Inspection + Verification)

### 🆕 `CC` — Current Default Mirror (Ollama Quota Burn)
- **Model:** `deepseek-v4-pro`
- **Primary:** ollama-cloud (⚠️ phase-out — DEPLETE ASAP)
- **Fallback:** openrouter (`deepseek/deepseek-v4-flash:free`)
- **Character:** Full toolset, 9999 max turns, no tool enforcement
- **Purpose:** Mirror of current `~/.hermes/config.yaml`. Burns Ollama Cloud quotas on general tasks until refund/service cut-off.

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