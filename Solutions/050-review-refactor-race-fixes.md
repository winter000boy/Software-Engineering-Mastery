# #050 — Solution: Review Previous Day's Build; Refactor Weak Spots

**Date:** 2026-10-12  
**Problem:** [050-review-refactor-race-fixes.md](../problems/050-review-refactor-race-fixes.md)

> Verified on JDK 21.0.10, Apple M4 (4 performance + 6 efficiency cores). Every number below is measured: detection rates over 200 attempts, JMH throughput, and the hypothesis test. After the refactor: **22/22 tests pass**, **zero `-Xlint:all` warnings**.

---

## Part 1 — Review Findings

| # | Check | Verdict | Finding |
|---|-------|:---:|---------|
| R1 | Tests can fail | ❌ **High** | Run against the racy implementations 200 times, the counter test caught the race only **77%** of the time and the seat test only **47%**. Re-introducing the seat bug would pass CI about half the time |
| R2 | Why weak? | 🔍 | Hypothesis tested; **partly refuted** (below) |
| R3 | Performance under contention | ⚠️ **Medium** | #049's counter choice (`LongAdder`) is confirmed. The CAS seat fix is **4× slower than the lock** at 8 threads, and erratic |
| R4 | API hygiene | ❌ **Medium** | Deliberately broken classes were **public members of the production interfaces** (`HitCounter.Racy`, `SeatInventory.Racy`, …), plus a demo harness in main sources |
| R5 | Documented semantics | ⚠️ Low | `LongAdder.get()` isn't an atomic snapshot while writers run; undocumented |
| R6 | Locking discipline | ✅ | Private locks, getters locked, no I/O under locks |
| R7 | Lint | ✅ | Clean |

### R1 — "Testing the tests" (mutation-style)

### `TestTheTests.java (final)`

```java
package com.mastery.conc.race;

/**
 * Mutation-style check: run each contract's concurrency test against the RACY implementation
 * many times. A good test should fail (detect the bug) on every attempt.
 */
public final class TestTheTests {
    interface Check { void run() throws Exception; }

    static void detectionRate(String name, int attempts, Check check) {
        int detected = 0;
        for (int i = 0; i < attempts; i++) {
            try {
                check.run();
            } catch (AssertionError e) {
                detected++;
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
        System.out.printf("%-48s detected the race in %3d/%d runs (%.0f%%)%n", name, detected, attempts, 100.0 * detected / attempts);
    }

    public static void main(String[] args) {
        int attempts = 200;
        HitCounterContractTest counter = new HitCounterContractTest() {
            @Override HitCounter newCounter() { return new RacyImplementations.RacyHitCounter(); }
        };
        SeatInventoryContractTest seats = new SeatInventoryContractTest() {
            @Override SeatInventory newInventory(int c) { return new RacyImplementations.RacySeatInventory(c); }
        };
        WordCounterContractTest hashMap = new WordCounterContractTest() {
            @Override WordCounter newCounter() { return new RacyImplementations.RacyHashMapWordCounter(); }
        };
        WordCounterContractTest getThenPut = new WordCounterContractTest() {
            @Override WordCounter newCounter() { return new RacyImplementations.GetThenPutWordCounter(); }
        };
        detectionRate("HitCounter.noLostUpdatesUnderContention", attempts, counter::noLostUpdatesUnderContention);
        detectionRate("SeatInventory.neverOversellsUnderContention", attempts, seats::neverOversellsUnderContention);
        detectionRate("WordCounter(HashMap).noLostCountsUnderContention", attempts, hashMap::noLostCountsUnderContention);
        detectionRate("WordCounter(CHM get+put).noLostCountsUnderContention", attempts, getThenPut::noLostCountsUnderContention);
        detectionRate("SeatInventory.sellsExactlyCapacitySequentially", attempts, seats::sellsExactlyCapacitySequentially);
    }
}
```

**Before (the #049 tests, one concurrent round each):**

```text
HitCounter.noLostUpdatesUnderContention          detected the race in 153/200 runs (77%)
SeatInventory.neverOversellsUnderContention      detected the race in  94/200 runs (47%)
WordCounter(HashMap).noLostCountsUnderContention detected the race in 200/200 runs (100%)
WordCounter(CHM get+put).noLostCountsUnderContention detected the race in 200/200 runs (100%)
SeatInventory.sellsExactlyCapacitySequentially   detected the race in   0/200 runs (0%)
```

(The sequential test is *supposed* to miss races; it checks semantics. It's listed as a sanity baseline.)

### R2 — Why only 47–77%? A hypothesis, tested

**Hypothesis:** "Once the JIT compiles the loop, a racy `count++` loop collapses into one read-add-write per thread, so the threads' windows rarely overlap and the result looks correct."

**Experiment:** run the racy counter (8 threads × 100 000) many times in one JVM, and count how often the result comes out *correct*, in the first half (cold) vs the second half (compiled), plus an interpreter-only run:

```text
JIT:    rounds 400: racy result was CORRECT in 26/200 early rounds, 6/200 late rounds
-Xint:  rounds  40: racy result was CORRECT in  0/20 early rounds,  0/20 late rounds
```

**Verdict: the hypothesis is refuted as stated.** False passes are **more** common **early** (warm-up), not late. Under the interpreter they never happen. A more plausible explanation is that during warm-up threads run at very different speeds (some still interpreted, some running freshly compiled code), so one thread may finish its whole loop before another really gets going, and the racy windows don't overlap. That fits the data but wasn't proven further. **The practical conclusion doesn't depend on the explanation:** a single round of a concurrency test is a probabilistic detector.

---

## Part 2 — Refactors

### Fix 1: repeat the concurrent scenario (R1)

### `HitCounterContractTest.java (final)`

```java
package com.mastery.conc.race;

import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

abstract class HitCounterContractTest {
    abstract HitCounter newCounter();

    @Test
    void countsSequentially() {
        HitCounter c = newCounter();
        for (int i = 0; i < 1_000; i++) c.increment();
        assertEquals(1_000, c.get());
    }

    /** One round detects a race only sometimes (measured in #050), so repeat it with fresh state. */
    static final int ROUNDS = 25;

    @Test
    void noLostUpdatesUnderContention() throws InterruptedException {
        int threads = 8, per = 100_000;
        for (int round = 0; round < ROUNDS; round++) {
            HitCounter c = newCounter();
            ConcurrencyTestSupport.runConcurrently(threads, () -> { for (int i = 0; i < per; i++) c.increment(); });
            assertEquals((long) threads * per, c.get(), "round " + round);
        }
    }
}

class SynchronizedHitCounterTest extends HitCounterContractTest {
    @Override HitCounter newCounter() { return new HitCounter.Synchronized(); }
}

class AtomicHitCounterTest extends HitCounterContractTest {
    @Override HitCounter newCounter() { return new HitCounter.Atomic(); }
}

class AdderHitCounterTest extends HitCounterContractTest {
    @Override HitCounter newCounter() { return new HitCounter.Adder(); }
}
```

### `SeatInventoryContractTest.java (final)`

```java
package com.mastery.conc.race;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.Test;

abstract class SeatInventoryContractTest {
    abstract SeatInventory newInventory(int capacity);

    @Test
    void sellsExactlyCapacitySequentially() {
        SeatInventory inv = newInventory(3);
        assertTrue(inv.tryBook());
        assertTrue(inv.tryBook());
        assertTrue(inv.tryBook());
        assertFalse(inv.tryBook());
        assertEquals(3, inv.booked());
    }

    /** One round detects a race only sometimes (measured in #050), so repeat it with fresh state. */
    static final int ROUNDS = 25;

    @Test
    void neverOversellsUnderContention() throws InterruptedException {
        int capacity = 1_000;
        for (int round = 0; round < ROUNDS; round++) {
            SeatInventory inv = newInventory(capacity);
            AtomicInteger sold = new AtomicInteger();
            ConcurrencyTestSupport.runConcurrently(8, () -> {
                for (int i = 0; i < 1_000; i++) if (inv.tryBook()) sold.incrementAndGet();
            });
            assertEquals(capacity, sold.get(), "tickets handed out, round " + round);
            assertEquals(capacity, inv.booked(), "seats recorded, round " + round);
        }
    }
}

class SynchronizedSeatInventoryTest extends SeatInventoryContractTest {
    @Override SeatInventory newInventory(int capacity) { return new SeatInventory.Synchronized(capacity); }
}

class CasSeatInventoryTest extends SeatInventoryContractTest {
    @Override SeatInventory newInventory(int capacity) { return new SeatInventory.Cas(capacity); }
}
```

**After:**

```text
HitCounter.noLostUpdatesUnderContention          detected the race in 200/200 runs (100%)
SeatInventory.neverOversellsUnderContention      detected the race in 200/200 runs (100%)
WordCounter(HashMap).noLostCountsUnderContention detected the race in 200/200 runs (100%)
WordCounter(CHM get+put).noLostCountsUnderContention detected the race in 200/200 runs (100%)
```

The arithmetic: if one round detects the race with probability p = 0.47, then 25 independent rounds miss it with probability (1 − 0.47)²⁵ ≈ **1.3 × 10⁻⁷**. The cost is about **0.6 s** more per counter test class. A fresh instance per round matters: a single instance reused across rounds would let one round's damage mask (or fake) another's.

### Fix 2: move broken code out of production sources (R4)

The racy implementations were public nested classes of the production interfaces, so any caller could write `new HitCounter.Racy()`. They're now package-private in **test sources**, along with the harness:

### `RacyImplementations.java (test sources)`

```java
package com.mastery.conc.race;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Deliberately BROKEN implementations, kept in TEST sources only: they exist to reproduce the
 * races (RaceHarness) and to prove the contract tests detect them (TestTheTests). Moved out of
 * the production interfaces in #050 so nobody can depend on them by accident.
 */
final class RacyImplementations {
    private RacyImplementations() { }

    /** ❌ count++ is read → add → write; concurrent increments overwrite each other. */
    static final class RacyHitCounter implements HitCounter {
        private long count;
        @Override public void increment() { count++; }
        @Override public long get() { return count; }
    }

    /** ❌ Two threads can both see booked == capacity - 1 and both book. */
    static final class RacySeatInventory implements SeatInventory {
        private final int capacity;
        private int booked;
        RacySeatInventory(int capacity) { this.capacity = capacity; }
        @Override public boolean tryBook() {
            if (booked < capacity) {          // CHECK
                booked++;                     // ACT (and itself a read-modify-write)
                return true;
            }
            return false;
        }
        @Override public int booked() { return booked; }
        @Override public int capacity() { return capacity; }
    }

    /** ❌ HashMap is not thread-safe at all: lost entries, wrong counts, corrupted structure. */
    static final class RacyHashMapWordCounter implements WordCounter {
        private final Map<String, Integer> counts = new HashMap<>();
        @Override public void add(String w) { counts.put(w, counts.getOrDefault(w, 0) + 1); }
        @Override public Map<String, Integer> snapshot() { return new HashMap<>(counts); }
    }

    /** ❌ A thread-safe map used with check-then-act: each call is atomic, the SEQUENCE is not. */
    static final class GetThenPutWordCounter implements WordCounter {
        private final Map<String, Integer> counts = new ConcurrentHashMap<>();
        @Override public void add(String w) { counts.put(w, counts.getOrDefault(w, 0) + 1); }
        @Override public Map<String, Integer> snapshot() { return new HashMap<>(counts); }
    }
}
```

`RaceHarness` moved to test sources as well; it still runs from `target/test-classes`.

### Fix 3: document the semantics and the measured choice (R5)

### `HitCounter.java (final)`

```java
package com.mastery.conc.race;

import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

/**
 * Counts events from many threads. Race #1: read-modify-write (lost updates).
 *
 * <p>Choosing an implementation (measured in #050, 8 threads on one counter):
 * {@link Adder} for write-heavy shared counters (~1 700 ops/µs), {@link Atomic} when uncontended
 * or when an exact atomic read is needed, {@link Synchronized} when the count is part of a larger
 * invariant guarded by the same lock.
 */
public interface HitCounter {
    void increment();

    /**
     * The current count. Exact once all increments have completed (e.g. after join). While
     * increments are still running, {@link Adder#get()} is NOT an atomic snapshot: it sums the
     * cells one by one and can miss increments that happen during the sum.
     */
    long get();

    /** ✅ Fix 1: one lock guards every access (reads too, for visibility). */
    final class Synchronized implements HitCounter {
        private final Object lock = new Object();
        private long count;                                     // guarded by lock
        @Override public void increment() { synchronized (lock) { count++; } }
        @Override public long get() { synchronized (lock) { return count; } }
    }

    /** ✅ Fix 2: lock-free CAS on one variable. */
    final class Atomic implements HitCounter {
        private final AtomicLong count = new AtomicLong();
        @Override public void increment() { count.incrementAndGet(); }
        @Override public long get() { return count.get(); }
    }

    /** ✅ Fix 3: striped counters summed on read: scales under heavy write contention. */
    final class Adder implements HitCounter {
        private final LongAdder count = new LongAdder();
        @Override public void increment() { count.increment(); }
        @Override public long get() { return count.sum(); }    // not an atomic snapshot while writers run
    }
}
```

---

## R3 — Contention Benchmark (JMH)

Every benchmark thread calls the **same** shared instance (`@State(Scope.Benchmark)`). Throughput is in **operations per microsecond, summed over all threads** (higher is better). JMH 1.37, 3 × 1 s warm-up, 5 × 1 s measurement, 1 fork, `-t 1 / 4 / 8`.

| | 1 thread | 4 threads | 8 threads |
|---|---:|---:|---:|
| Counter: `synchronized` | 445 ± 3 | 28 ± 1 | 35 ± 1 |
| Counter: `AtomicLong` | **606 ± 10** | 139 ± 1 | 42 ± 4 |
| Counter: `LongAdder` | 244 ± 1 | **872 ± 5** | **1 716 ± 6** |
| Seats: `synchronized` | **450 ± 6** | 29 ± 9 | **31 ± 10** |
| Seats: CAS loop | 252 ± 3 | 27 ± 17 | 7 ± 10 |

What this shows:

1. **Contention on one variable collapses throughput.** `AtomicLong` goes 606 → 42 ops/µs from 1 to 8 threads (−93%): every CAS fights over the same cache line, and losers retry. `synchronized` collapses similarly.
2. **`LongAdder` scales *up* with threads (244 → 1 716).** Each thread updates its own cell (striping), so there's no shared hot spot. Uncontended it's slower than `AtomicLong` (244 vs 606) because of the extra indirection. #049's choice for a write-heavy shared counter is **confirmed**.
3. **The CAS loop for seats is the worst option under contention** (7 ± 10 ops/µs at 8 threads, with an error bar bigger than the mean). The check-then-CAS retry loop does more work per failed attempt than a counter's single CAS, so threads repeatedly lose and retry. The `synchronized` version is **~4× faster** and much more stable. #049's choice of a lock for seats is **confirmed**, now with data rather than intuition.
4. The 4- and 8-thread lock and CAS numbers have **large error bars** (±9 to ±17): contended performance is erratic, which is a cost in itself (unpredictable latency).

**Decision record:**

| Race | Ship | Evidence |
|------|------|----------|
| Hit counter | `LongAdder` | 1 716 vs 42 ops/µs at 8 threads; reads are rare |
| Seats | `synchronized` | ~4× the CAS loop at 8 threads, more stable; check-then-act needs atomicity across the check and the act anyway |
| Word counts | `ConcurrentHashMap.merge` | Atomic per key (not benchmarked; contention spreads across keys) |

### `FixBenchmark.java`

```java
package com.mastery.conc.race;

import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;

/**
 * Throughput of the race fixes when ALL benchmark threads hammer ONE shared instance
 * (Scope.Benchmark). Run with -t 1, -t 4, -t 8 to vary contention.
 */
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class FixBenchmark {

    @State(Scope.Benchmark)
    public static class Counters {
        final HitCounter sync = new HitCounter.Synchronized();
        final HitCounter atomic = new HitCounter.Atomic();
        final HitCounter adder = new HitCounter.Adder();
    }

    @State(Scope.Benchmark)
    public static class Seats {
        SeatInventory sync;
        SeatInventory cas;

        @Setup(org.openjdk.jmh.annotations.Level.Iteration)
        public void reset() {                       // effectively unlimited seats: measures the booking path
            sync = new SeatInventory.Synchronized(Integer.MAX_VALUE);
            cas = new SeatInventory.Cas(Integer.MAX_VALUE);
        }
    }

    @Benchmark public void counterSynchronized(Counters s) { s.sync.increment(); }
    @Benchmark public void counterAtomicLong(Counters s)   { s.atomic.increment(); }
    @Benchmark public void counterLongAdder(Counters s)    { s.adder.increment(); }
    @Benchmark public boolean seatsSynchronized(Seats s)   { return s.sync.tryBook(); }
    @Benchmark public boolean seatsCasLoop(Seats s)        { return s.cas.tryBook(); }
}
```

---

## Final State

```text
$ mvn test                         (whole Week 3 project)
Tests run: 22, Failures: 0, Errors: 0, Skipped: 0
$ javac -Xlint:all (main sources)  → no warnings
$ grep -r Racy src/main            → nothing (broken code lives only in test sources)
```

---

## Key Takeaways

1. **Test your concurrency tests against the bug they guard against.** A single round caught the races only 47–77% of the time.
2. **Repeat rounds with fresh state.** Detection goes to 100% (miss probability ~10⁻⁷) for under a second of test time.
3. **Test hypotheses, and report refutations honestly.** "The JIT collapses the loop late" was wrong; false passes cluster during warm-up.
4. **Measure contention before choosing a fix:** `LongAdder` scales (1 716 ops/µs at 8 threads), single-variable CAS and locks collapse, and a **CAS retry loop can lose to a plain lock** under contention.
5. Keep **demo/broken code out of production APIs**, and document subtle semantics (`LongAdder.sum` isn't a snapshot).
