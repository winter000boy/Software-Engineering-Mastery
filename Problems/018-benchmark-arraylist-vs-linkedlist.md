# #018 — Benchmark ArrayList vs LinkedList Operations

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 18 — P1 Java Core · Collections & Generics  
**Topics:** microbenchmarking, JMH, JIT warm-up, dead-code elimination, Big-O vs real performance, CPU cache locality, memory footprint (JOL)  
**Background reading:** #017 (List section), #020 (internals)

---

## Overview

Textbooks say *"LinkedList is O(1) for insertion, so use it when you insert a lot."* Your job is to **measure** whether that's true on real hardware, and to explain every result from how the two classes are built.

You'll use **JMH** (Java Microbenchmark Harness, from the OpenJDK team), because hand-written `System.nanoTime()` loops give wrong answers on the JVM.

---

## Part 1 — Why Not Just `System.nanoTime()`?

Before writing any benchmark, explain in your own words how each of these would distort a naive timing loop:

1. **JIT warm-up** (interpreter → C1 → C2 tiers)
2. **Dead-code elimination** (the result of the loop is never used)
3. **Constant folding / loop hoisting**
4. **GC pauses** during the measurement
5. **On-stack replacement (OSR)** of a long-running loop in `main`
6. Measuring **one run** in **one JVM**

Then name the JMH feature that addresses each one.

---

## Part 2 — Set Up JMH

1. Create a Maven project that produces a runnable `benchmarks.jar` (JMH's `org.openjdk.jmh.Main` as the entry point).
2. Configure: average-time mode, microseconds, 3 warm-up iterations × 1 s, 5 measurement iterations × 1 s, 1 fork, a fixed heap (`-Xms2g -Xmx2g`).
3. Parameterise the benchmark on:
   - `size` ∈ {1 000, 100 000}: elements already in the list
   - `impl` ∈ {`ArrayList`, `LinkedList`}

---

## Part 3 — Benchmarks to Write

For each one, **predict first** (write down the Big-O for each implementation and which will win), then measure.

| # | Benchmark | What it does |
|---|-----------|--------------|
| B1 | `addLast` | Build a new list of `size` elements with `add(e)` |
| B2 | `getRandom` | 1 000 `get(i)` calls at random indexes (fixed seed) |
| B3 | `iterate` | Sum all elements with for-each |
| B4 | `addLastRemoveFirst` | Queue usage: 1 000 × (`add(e)` then `remove(0)`) |
| B5 | `addFirstRemoveFirst` | 1 000 × `add(0, e)`, then 1 000 × `remove(0)` |
| B6 | `insertMiddleViaIterator` | Position a `ListIterator` at `size/2`, insert 1 000 elements through it, then remove them through it |
| B7 | `containsMissing` | `contains(-1)`, a full scan with no match |
| B8 | `indexLoop` | `for (int i = 0; i < list.size(); i++) list.get(i)`, **only at size 1 000** (explain why) |

Rules:

- Each benchmark must **return** its result or feed it to a `Blackhole`.
- Benchmarks that mutate the shared list must **restore its size** so later invocations measure the same thing.
- Use a **fixed random seed** so both implementations read the same indexes.

---

## Part 4 — Memory Footprint

Using **JOL** (`org.openjdk.jol:jol-core`), measure the retained heap of `ArrayList`, `LinkedList` and `ArrayDeque` each holding the same 100 000 `Integer` objects. Report:

- total bytes
- bytes of **structure** (excluding the `Integer`s themselves)
- overhead per element

Explain the per-element overhead from the object layout (object header, compressed references, the `Node` fields).

---

## Part 5 — Analyse

1. Put all results in one table: operation × size × impl, plus the ratio LinkedList/ArrayList.
2. For every row where the result **disagrees with the Big-O prediction**, explain why.
3. Explain **cache locality**: why does traversing 100 000 elements take different times when both are O(n)?
4. B4 is a queue workload. Add a third implementation, `ArrayDeque`, and compare (hint: `offerLast`/`pollFirst`).
5. Write the final recommendation: when, if ever, would you choose `LinkedList`?

---

## Constraints

- JMH 1.37, Java 21.
- Record your hardware (CPU, cores) and JDK: benchmark numbers mean nothing without them.
- Close heavy apps while measuring. Don't compare runs from different machines.
- Look at the *error* column (± 99.9% CI). If the error is larger than the difference, you haven't shown a difference.

---

## Acceptance Criteria

- [ ] Part 1: all 6 pitfalls explained + the JMH remedy for each
- [ ] JMH project builds `benchmarks.jar`; all 8 benchmarks run for both sizes and both impls
- [ ] Predictions written **before** the run
- [ ] Results table with ratios; every surprise explained
- [ ] JOL footprint measured and explained
- [ ] `ArrayDeque` compared on the queue workload
- [ ] Final recommendation written

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
