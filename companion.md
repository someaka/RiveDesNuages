# Companion Notes — Anecdotes & Operational Wisdom

> Companion file to `notes.md`. Captures reflections, hard-won truths,
> and patterns that don't fit in structured endpoint documentation.
> These are the things the next agent needs to understand *why* we built it this way.
> Copied from ~/Desktop/agenda/companion.md — mirror for RiveDesNuages folder.

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
- Additional Nous portal endpoints beyond qwen3.6-plus
- Any endpoints from the ZenMux full catalog once evaluation completes
- Whether local Qwen 3.6 variants graduate from proving ground to production

---

## The Moonshots MIMO Question

MIMO pricing: $100/mo standard, $77 new-customer, potentially $70/mo if Discord answer confirms.
User won't wire MIMO into automated routing — it's strictly break-glass manual use.
The priming protocol matters enormously: wrong priming = "horrifying demon." Correct priming = catches its own mistakes.
**Until the priming protocol is documented, MIMO stays manual-only.**

---

*This file grows as conversations happen. It's the institutional memory that survives agent rotations.*