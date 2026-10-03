# #036 — Document & Review: LRU Cache

**Date:** 2026-10-03  
**Type:** Project — Phase 5 of 5  
**Roadmap row:** 36 — P1 Java Core · Collections & Generics  
**Prerequisite:** **#032–#035**

---

## Goal

Turn the working code into something another engineer could pick up: documentation, recorded decisions, an honest list of limitations, and a review of the project against its own requirements.

---

## Tasks

1. **README** for the library:
   - what it is, and when to use which implementation
   - a quick-start code sample
   - the behavioural contract in plain words (what counts as a "use", eviction rules, listener semantics including exceptions)
   - performance summary (from #035) and hit-rate guidance
   - build & test commands
2. **Design decisions record**: one row per significant decision (composition over inheritance, `Optional` returns, null rejection, sentinels, decorator for thread safety, notify-after-mutation), each with context, decision, alternatives and consequences.
3. **Requirements traceability**: map every FR/NFR from #032 to the test(s) or benchmark that proves it. Mark anything **not** met.
4. **Code review** against this checklist: contracts documented; invariants listed; complexity documented; generics and wildcards correct; immutability where possible; no leaked internals; exception behaviour defined; tests independent and deterministic; benchmarks reproducible.
5. **Limitations & future work**: what would it take to make this production-grade? Compare with Caffeine at a conceptual level (expiry, weights, admission policy, concurrency).
6. **Retrospective**: what went well, what went wrong (including any bugs found late), and what you'd do differently.

---

## Acceptance Criteria

- [ ] README complete, with a working quick-start
- [ ] ≥ 6 decisions recorded with alternatives
- [ ] Every FR/NFR traced to evidence
- [ ] Review checklist completed, with findings
- [ ] Limitations and a Caffeine comparison
- [ ] Retrospective with at least one "what went wrong"

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
