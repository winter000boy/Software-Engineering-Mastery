# #046 — Producer-Consumer with `wait`/`notify`

**Date:** 2026-10-12  
**Type:** Build  
**Roadmap row:** 46 — P1 Java Core · Concurrency & Multithreading I  
**Topics:** monitors & wait sets, condition predicates, `wait()` in a loop, `notify` vs `notifyAll`, bounded buffers, interruption, testing blocking code without `sleep`  
**Background:** #045 §6 (monitor = lock + wait set); leads into the blocking-queue project (rows 57–61)

---

## Overview

Build a **bounded buffer**: a fixed-capacity FIFO that producers `put` into and consumers `take` from, where `put` **blocks while full** and `take` **blocks while empty**. Use **only** `synchronized` + `wait()` / `notifyAll()`. No `java.util.concurrent` in the implementation.

Then **break it on purpose** in the two classic ways and measure how often each breaks.

---

## Part 1 — `BoundedBuffer<E>`

```java
public final class BoundedBuffer<E> {
    public BoundedBuffer(int capacity)                 // capacity > 0
    public void put(E e) throws InterruptedException   // blocks while full; rejects null
    public E take() throws InterruptedException        // blocks while empty
    public int size()
    public int capacity()
}
```

Requirements:

1. Storage is a **circular array** (`head` + `count`), with O(1) `put`/`take` and no shifting. Null out taken slots.
2. Use a **private lock object**, not `this`. Explain why that matters especially for `wait`/`notify`.
3. Write the **invariant** and the two **condition predicates** in the class Javadoc (JCIP §14.1 style).
4. Waiting must be **interruptible** (`InterruptedException` propagates).
5. Explain *precisely* why the wait must be in a `while` loop and why you use `notifyAll`.

## Part 2 — Tests (no `Thread.sleep`!)

- FIFO order, including wrap-around of the circular array
- `put` on a full buffer **blocks** (assert the producer reaches `Thread.State.WAITING`), then completes after a `take`
- `take` on an empty buffer blocks, and **is interruptible**
- 6 producers × 50 000 unique items, 4 consumers, capacity 8: every item is delivered **exactly once** (no loss, no duplicates)
- With a single consumer, each producer's items arrive **in order**
- Argument validation
- Every concurrent test has a timeout, so a deadlock fails instead of hanging the build

## Part 3 — Break It and Measure

Write two broken variants and a harness that runs 4 producers + 4 consumers × 2 000 items through a **capacity-1** buffer, 50 times per variant, classifying each run as **OK**, **EXCEPTION** (an invariant check failed) or **HANG** (no progress within 2 s):

1. **`if` instead of `while`** around `wait()` (add a check that throws if the predicate is false after waking)
2. **`notify()` instead of `notifyAll()`** (keeping the `while`)

Predict the outcome first. Then explain each failure with a concrete interleaving of threads.

## Part 4 — Graceful Shutdown

How do consumers know there will be no more items? Describe the **poison pill** technique (and its pitfall with several consumers), and compare it with interrupting the consumers.

---

## Acceptance Criteria

- [ ] `BoundedBuffer` with a circular array, a private lock, documented invariant/predicates, `while` + `notifyAll`
- [ ] All Part 2 tests green, with no `sleep` in them
- [ ] Part 3 measured: OK / EXCEPTION / HANG counts per variant, with a failure interleaving for each
- [ ] Poison pill vs interruption explained
- [ ] Zero `-Xlint:all` warnings

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
