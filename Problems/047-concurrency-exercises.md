# #047 — Solve 2–3 Exercises on Concurrency

**Date:** 2026-10-12  
**Type:** Exercise  
**Roadmap row:** 47 — P1 Java Core · Concurrency & Multithreading I · *Flex*  
**Topics:** turn-taking with condition predicates, `wait`/`notifyAll`, termination conditions, building a synchronizer, timed waits, happens-before through your own synchronizer  
**Background:** #045, #046

---

## Overview

Three classic coordination exercises. Use **only** `synchronized`, `wait`, `notifyAll` and `Thread`: no `java.util.concurrent` in the solutions (tests may use `AtomicInteger` and friends).

---

## Exercise 1 — Ping-Pong

Two threads, `ping` and `pong`, print `"ping"` and `"pong"` **strictly alternately**, N times each, always starting with `"ping"`, **even if the `pong` thread is started first**.

```
ping pong ping pong ping pong ...
```

Return the observed sequence as a `List<String>` so a test can verify it.

## Exercise 2 — Round-Robin Printer

K threads print the numbers 1..max in order, where thread *i* prints exactly the numbers n with `(n − 1) % K == i`:

```
K = 3, max = 7:  T0:1  T1:2  T2:3  T0:4  T1:5  T2:6  T0:7
```

- Start the threads in **reverse** order. Output must be unaffected.
- **All threads must terminate**, including when K > max (some threads print nothing).

## Exercise 3 — Build Your Own `CountDownLatch`

```java
public final class SimpleCountDownLatch {
    public SimpleCountDownLatch(long count)            // count >= 0
    public void countDown()                            // at 0, release ALL waiters; extra calls do nothing
    public void await() throws InterruptedException    // block until count == 0
    public boolean await(long timeout, TimeUnit unit) throws InterruptedException   // false on timeout
    public long getCount()
}
```

- Waiters released **only** when the count hits 0, and **all** of them.
- One-shot: it never re-arms.
- **Memory guarantee:** writes made before `countDown()` must be visible after `await()` returns. Explain which mechanism provides this.
- Timed `await` must handle early returns from `wait(ms)` (spurious wake-ups, notifications) by recomputing the remaining time.

---

## Tests (required)

- Ping-pong: 1 000 rounds; the exact alternation verified at every position; **repeat the test 20 times**
- Round-robin: K = 4, max = 500, every line verified, repeated 20 times; K = 8, max = 3 terminates
- Latch: 5 waiters stay blocked through two `countDown`s and are all released by the third; an extra `countDown` keeps the count at 0; the timed `await` times out after ≥ 50 ms and succeeds once open; a happens-before test (a plain field written before `countDown` is read correctly after `await`, 1 000 rounds)
- Every concurrent test under `assertTimeoutPreemptively`

---

## Acceptance Criteria

- [ ] All three implemented with intrinsic locks only
- [ ] Each has a documented condition predicate (what each thread waits for)
- [ ] All tests green, including the repeated ones
- [ ] The happens-before mechanism of your latch explained

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
