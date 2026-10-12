# #043 — Create & Coordinate Threads

**Date:** 2026-10-12  
**Type:** Build  
**Roadmap row:** 43 — P1 Java Core · Concurrency & Multithreading I  
**Topics:** `Thread`, `Runnable`, `join`, thread confinement, work partitioning & load balance, cooperative cancellation with interrupts, propagating worker failures, measuring speedup  
**Background:** #042

---

## Overview

Use **only** `Thread`, `Runnable`, `join()` and `interrupt()` (plus at most one `AtomicLong`) to run work on several threads, collect the results, cancel early, and surface failures. Executors and `CompletableFuture` come later (rows 67+). Building it by hand first shows what they do for you.

---

## Task 1 — Parallel Prime Count

Count the primes in `[0, limit)` with `N` platform threads, using a deliberately CPU-bound `isPrime` (trial division up to √n, so larger numbers cost more).

1. Write `countSequential(from, to)` as the reference.
2. Write `count(limit, threads, strategy)` with two partitioning strategies:
   - **BLOCK:** thread *i* takes the contiguous range `[i·limit/N, (i+1)·limit/N)`
   - **STRIPED:** thread *i* takes `i, i+N, i+2N, …`
3. Each worker must write its result **only to its own slot** of a `long[] partial`. The coordinator `join()`s every worker and then sums. Explain why this needs **no lock**, and why `join()` makes the reads safe.
4. Test: both strategies match the sequential count for N ∈ {1, 2, 3, 7, 10, 16} (π(200 000) = 17 984).

## Task 2 — Measure Speedup

For `limit = 10 000 000` and N ∈ {1, 2, 4, 6, 8, 10, 16}, measure the wall time of each strategy (best of 3 after a warm-up) and the speedup over 1 thread. Also record the **per-thread busy time** of BLOCK with 4 threads.

- Predict first: which strategy balances the load better, and why?
- Then explain the results you actually got. **At least one result should surprise you.** Find the cause.
- Propose and implement a third strategy that balances **automatically**, whatever the data looks like.

## Task 3 — Cancel Early: "Find Any"

`findAny(from, to, threads, predicate)` returns **any** number in range matching the predicate. As soon as one worker finds a match, **all the other workers must stop**.

- Use `interrupt()` for cancellation; workers check `isInterrupted()`.
- The winner must wake the coordinator, which is blocked in `join()`.
- After returning, **no worker may still be running**, and the coordinator's own interrupt flag must **not** be left set.
- A real external interrupt of the coordinator (no match found) must still propagate as `InterruptedException`.

## Task 4 — Don't Lose Worker Failures

`runAll(List<Runnable> tasks)` runs each task on its own thread and, if any of them threw, throws **one** exception in the caller that carries **all** the failures (as suppressed exceptions).

---

## Constraints

- No `ExecutorService`, `CompletableFuture`, parallel streams or `ForkJoinPool`.
- No `Thread.sleep` for coordination, in code or tests.
- Tests must be deterministic. The cancellation test needs a timeout (`assertTimeoutPreemptively`) so a bug fails the test instead of hanging it.

---

## Acceptance Criteria

- [ ] Both strategies (+ your third) are correct for all N in the test
- [ ] Speedup table + per-thread busy times measured and explained, including the surprise
- [ ] A third, self-balancing strategy, measured
- [ ] `findAny` stops all workers, returns the match, leaks no interrupt flag
- [ ] `runAll` surfaces every worker failure
- [ ] Zero `-Xlint:all` warnings

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
