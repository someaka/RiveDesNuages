# Companion Notes — Anecdotes & Operational Wisdom

> Companion file to `notes.md`. Captures reflections, hard-won truths,
> and patterns that don't fit in structured endpoint documentation.
> These are the things the next agent needs to understand *why* we built it this way.
> Copied from ~/Desktop/agenda/companion.md — mirror for RiveDesNuages folder.
> Last update: 2026-05-19 — free profile resurrected with 4-layer fallback, Ollama Cloud phase-out (FA→FO), CC profile added.

---

## The Verification Problem (Stage 11 Is Not a Stage)

The kanban pipeline was designed as a linear flow: Investigate → Specs → Plan → Verify → Execute → ... → Accept.

**That's not how it works.**

"Accept" is not a boolean gate. It's a negotiation between what the agents produced and what actually runs in production. The pipeline collapses at Stage 11 into a recursive mess:

```
Execute → Verify → Fix → Verify → Fix → Verify → ... → user stares at it → fixes it themselves
```

Agents freeze at the threshold where output *looks structurally correct* but doesn't *actually work*. They can't distinguish "technically valid" from "works in production." So the loop either:
- Spins forever with diminishing returns
- Converges on something the user has to fix manually anyway

**This is not a failure of our pipeline design — it's the actual workflow in spring 2026.** The pipeline needs to model Stage 11 as a **re-entry point** with its own sub-kanban, not a terminal state.

### Implication for Profile Design
- `inspector` (GLM5.1) is the verification workhorse, but it cannot be the *final* verifier
- Final verification is human. Always.
- The pipeline should budget for multiple `worker` → `inspector` oscillation loops
- Task complexity estimates should include expected rework cycles, not just first-pass execution

---

## The Leading-by-Example Pattern

User consistently observed: agents snap into place once they see a working example.

This means:
1. **The first iteration must be human-led.** Agents optimize for the path of least resistance — if the path isn't shown, they freeze.
2. **Agent value is in *scaling* a proven approach**, not in *discovering* one.
3. **Boilerplate / scaffolding is the highest-value task for agents.** Take something that works, replicate it, adapt it. Greenfield discovery is still human territory.
4. **"Replace me" is aspirational.** Right now, agents are force multipliers on tasks where success criteria are binary and verifiable. Anything requiring taste, judgment, or production debugging stays human.

---

---

## Endpoints People Forget

Every conversation, the user says "I'll remember a few more later." They never do all at once. The TBD list in the main notes is the living registry. New endpoints get added here first, promoted to the structured notes when they survive a full conversation cycle.

Current TBD candidates (unconfirmed):
~~- Additional Nous portal endpoints beyond qwen3.6-plus~~ ❌ **Moot** — qwen3.6-plus expired
- Any endpoints from the ZenMux full catalog once evaluation completes
- Whether local Qwen 3.6 variants graduate from proving ground to production

---

## The Great Free-Model Extinction Event (2026-05-15)

Two free models died on the same day:

1. **ring-2.6-1t (OpenRouter)** — was the `free` profile's primary for Stage 1 (Investigate). Outstanding TPS, zero cost. Gone.
2. **qwen3.6-plus (Nous)** — was the universal fallback for ALL four profiles. Every profile's safety net. Gone.

**What survived:** `deepseek-v4-flash` on nous (`deepseek/deepseek-v4-flash`). It's free *for now*, but so were the other two. The pattern is clear: **free offers on inference APIs are temporary by nature.** The only permanent $0 inference path is local models.

### Resurrection: Multi-Layer Free Fallback (2026-05-19)

The `free` profile was revived and hardened with a four-layer fallback chain:
1. `deepseek-v4-flash` on nous (primary — direct)
2. `deepseek/deepseek-v4-flash:free` on openrouter (fallback 1)
3. `deepseek-v4-flash` on opencode-go (fallback 2)
4. `stepfun/step-3.5-flash` on nous (last resort — proven in Hermes auxiliary tasks)

**Implications:**
- The `free` profile is **active** with four chances before it goes dark
- Current config.yaml default is `deepseek-v4-pro` on ollama-cloud (⚠️ phase-out — FA→FO escalation, burn quotas while they last)
- CC profile mirrors the default config to maximize Ollama quota depletion
- **Local models are the only reliable long-term $0 path** — qwen3.6 27B or 35B A3B benchmarking remains urgent
- The extinction pattern is clear: free API offers die. The fallback chain buys time.

---

## The Moonshots MIMO Question

MIMO pricing: $100/mo standard, $77 new-customer, potentially $70/mo if Discord answer confirms.
User won't wire MIMO into automated routing — it's strictly break-glass manual use.
The priming protocol matters enormously: wrong priming = "horrifying demon." Correct priming = catches its own mistakes.
**Until the priming protocol is documented, MIMO stays manual-only.**

---

*This file grows as conversations happen. It's the institutional memory that survives agent rotations.*