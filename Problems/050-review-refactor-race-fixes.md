# #050 — Review Previous Day's Build; Refactor Weak Spots

**Date:** 2026-10-12  
**Type:** Review + Refactor  
**Roadmap row:** 50 — P1 Java Core · Concurrency & Multithreading I · *Flex*  
**Reviews:** yesterday's build, the race-condition fixes and tests (#049)  
**Topics:** testing the tests (mutation-style), flaky detection of concurrency bugs, contention benchmarking with JMH, choosing between locks / CAS / striping with data, keeping demo code out of production APIs

---

## Overview

#049's fixes pass their tests. This review asks the harder questions: **would those tests catch the bug if it came back?** Are the chosen fixes actually the fast ones under contention? Is anything in the production API that shouldn't be?

---

## Part 1 — Review Checklist

Record *OK* or a *finding* for each:

| # | Question |
|---|----------|
| R1 | **Tests can fail:** run each concurrency contract test against the **racy** implementation many times (≥ 200). What's the detection rate? |
| R2 | If the detection rate isn't ~100%, **why**? Form a hypothesis and **test it** (e.g. compare early vs late rounds, JIT vs `-Xint`). |
| R3 | **Performance under contention:** benchmark the counter fixes (`synchronized`, `AtomicLong`, `LongAdder`) and the seat fixes (`synchronized`, CAS loop) with JMH at 1, 4 and 8 threads on **one shared instance**. Do #049's choices hold? |
| R4 | **API hygiene:** is there anything public in production code that exists only for demos or tests? |
| R5 | **Documented semantics:** is anything subtle undocumented (e.g. what `get()` means while writers are still running)? |
| R6 | Locking discipline: private locks, every access guarded, no I/O under locks |
| R7 | Lint: `-Xlint:all` clean? |

---

## Part 2 — Refactor

Fix every finding you decide to fix, and **prove** each fix:

- Test strength: re-measure the detection rate after strengthening (target: 100% in 200 attempts). Report the cost in test time.
- API: move whatever doesn't belong out of production sources; the build and all tests must stay green.
- Documentation: add Javadoc for the semantics and for the implementation choice, citing your measurements.

---

## Acceptance Criteria

- [ ] Detection rate measured before and after, for every concurrency test
- [ ] Hypothesis for weak detection stated **and tested**, with the result reported honestly (even if it's refuted)
- [ ] JMH results at 1/4/8 threads; fix choices confirmed or revised with data
- [ ] Racy/demo code removed from production sources
- [ ] Semantics documented
- [ ] All tests green, zero lint warnings

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
