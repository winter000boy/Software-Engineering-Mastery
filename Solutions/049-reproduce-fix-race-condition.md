# #049 — Solution: Reproduce, Then Fix, a Race Condition

**Date:** 2026-10-12  
**Problem:** [049-reproduce-fix-race-condition.md](../problems/049-reproduce-fix-race-condition.md)

> Verified on JDK 21.0.10, Apple M4 (10 cores): the harness output below is measured, and **all contract tests pass** for the fixed implementations. This is the code **as built in this phase**. Row 50's review later strengthens the tests and moves the racy classes out of production code.

---

## 1. Reproduction Results

```text
8 threads, 20 runs each, 10 cores
HitCounter  Racy (count++)   wrong in 20/20 runs; worst run lost 1,528,217 of 2,000,000 increments
HitCounter  Synchronized     wrong in  0/20 runs; worst run lost 0 of 2,000,000 increments
HitCounter  AtomicLong       wrong in  0/20 runs; worst run lost 0 of 2,000,000 increments
HitCounter  LongAdder        wrong in  0/20 runs; worst run lost 0 of 2,000,000 increments
Seats       Racy (check-act) wrong in 17/20 runs; worst run sold 4,651 tickets for 1,000 seats
Seats       Synchronized     wrong in  0/20 runs; worst run sold 1,000 tickets for 1,000 seats
Seats       CAS loop         wrong in  0/20 runs; worst run sold 1,000 tickets for 1,000 seats
WordCounter HashMap          wrong in 20/20 runs; worst run lost 149,012 of 800,000 counts
WordCounter CHM get-then-put wrong in 20/20 runs; worst run lost 130,356 of 800,000 counts
WordCounter CHM merge        wrong in  0/20 runs; worst run lost 0 of 800,000 counts
```

Every racy implementation was wrong in (almost) every run. These aren't rare corner cases under artificial delays; they're what plain, natural-looking code does on a 10-core machine.

---

## 2. Race 1 — Lost Updates

### `HitCounter.java (as built)`

```java
package com.mastery.conc.race;

import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

/** Counts events from many threads. Race #1: read-modify-write (lost updates). */
public interface HitCounter {
    void increment();
    long get();

    /** ❌ count++ is read → add → write; concurrent increments overwrite each other. */
    final class Racy implements HitCounter {
        private long count;
        @Override public void increment() { count++; }
        @Override public long get() { return count; }
    }

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

**The interleaving:**

```
count = 41
T1: read count → 41
T2: read count → 41
T1: write 41 + 1 = 42
T2: write 41 + 1 = 42        ← T1's increment is lost
```

`count++` is three steps (#048 §3). Up to 76% of the increments were lost in the worst run, because with 8 threads hammering one field, most read-add-write sequences overlap with another.

| Fix | How it works |
|-----|--------------|
| `synchronized` | Only one thread at a time runs `count++`; the lock's unlock→lock edge makes the value visible (#045). `get()` locks too, for visibility |
| `AtomicLong` | `incrementAndGet` = a hardware **compare-and-swap** loop: "set to v+1 only if it's still v", retrying on failure. Lock-free |
| `LongAdder` | Contended threads add to **separate cells** (striping); `sum()` adds the cells up. Far less contention on one cache line |

---

## 3. Race 2 — Check-Then-Act

### `SeatInventory.java (as built)`

```java
package com.mastery.conc.race;

import java.util.concurrent.atomic.AtomicInteger;

/** Sells a fixed number of seats. Race #2: check-then-act (overselling). */
public interface SeatInventory {
    /** Books one seat if any is left; returns whether it succeeded. */
    boolean tryBook();
    int booked();
    int capacity();

    /** ❌ Two threads can both see booked == capacity - 1 and both book. */
    final class Racy implements SeatInventory {
        private final int capacity;
        private int booked;
        public Racy(int capacity) { this.capacity = capacity; }
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

    /** ✅ Fix 1: the check and the act happen under one lock. */
    final class Synchronized implements SeatInventory {
        private final Object lock = new Object();
        private final int capacity;
        private int booked;                   // guarded by lock
        public Synchronized(int capacity) { this.capacity = capacity; }
        @Override public boolean tryBook() {
            synchronized (lock) {
                if (booked < capacity) {
                    booked++;
                    return true;
                }
                return false;
            }
        }
        @Override public int booked() { synchronized (lock) { return booked; } }
        @Override public int capacity() { return capacity; }
    }

    /** ✅ Fix 2: lock-free: retry a compare-and-set until it wins or the seats run out. */
    final class Cas implements SeatInventory {
        private final int capacity;
        private final AtomicInteger booked = new AtomicInteger();
        public Cas(int capacity) { this.capacity = capacity; }
        @Override public boolean tryBook() {
            while (true) {
                int current = booked.get();
                if (current >= capacity) {
                    return false;
                }
                if (booked.compareAndSet(current, current + 1)) {   // succeeds only if nobody changed it meanwhile
                    return true;
                }
                // lost the race: someone else booked in between → re-read and retry
            }
        }
        @Override public int booked() { return booked.get(); }
        @Override public int capacity() { return capacity; }
    }
}
```

**The interleaving (the obvious part):**

```
capacity 1000, booked = 999
T1: check 999 < 1000 ✓
T2: check 999 < 1000 ✓        ← both pass the check before either acts
T1: booked++ → 1000, sells a ticket
T2: booked++ → 1001, sells a ticket   ← oversold
```

**Why 4 651 tickets for 1 000 seats (4.6×), not just a few extra:** `booked++` *itself* is a lost-update race (Race 1). Lost increments keep `booked` **lower than the number of tickets actually sold**, so the `booked < capacity` check keeps passing long after the real sales exceed capacity. Two races compound: one hides the evidence of the other.

**CAS loop:** `compareAndSet(current, current + 1)` succeeds only if nobody changed `booked` since we read it. If someone did, we **re-read and re-check**, so a check and an act that succeed are guaranteed to have seen the same value. That's the general lock-free pattern for check-then-act on one variable.

---

## 4. Race 3 — Maps

### `WordCounter.java (as built)`

```java
package com.mastery.conc.race;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/** Counts words from many threads. Race #3: unsafe map + compound actions on a "safe" map. */
public interface WordCounter {
    void add(String word);
    Map<String, Integer> snapshot();

    /** ❌ HashMap is not thread-safe at all: lost entries, wrong counts, corrupted structure. */
    final class RacyHashMap implements WordCounter {
        private final Map<String, Integer> counts = new HashMap<>();
        @Override public void add(String w) { counts.put(w, counts.getOrDefault(w, 0) + 1); }
        @Override public Map<String, Integer> snapshot() { return new HashMap<>(counts); }
    }

    /** ❌ A thread-safe map used with check-then-act: each call is atomic, the SEQUENCE is not. */
    final class GetThenPutOnConcurrentMap implements WordCounter {
        private final Map<String, Integer> counts = new ConcurrentHashMap<>();
        @Override public void add(String w) { counts.put(w, counts.getOrDefault(w, 0) + 1); }
        @Override public Map<String, Integer> snapshot() { return new HashMap<>(counts); }
    }

    /** ✅ Fix: ConcurrentHashMap.merge performs the read-modify-write atomically per key. */
    final class Merge implements WordCounter {
        private final ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
        @Override public void add(String w) { counts.merge(w, 1, Integer::sum); }
        @Override public Map<String, Integer> snapshot() { return new HashMap<>(counts); }
    }
}
```

- **`HashMap`:** not thread-safe at all. Concurrent `put`s can lose entries, corrupt bucket chains during a resize, and leave `size` wrong (#020 §3.11).
- **`ConcurrentHashMap` + get-then-put: still broken** (lost 130 356 counts). Each *individual* call is thread-safe, but **the sequence** `getOrDefault` → compute → `put` isn't atomic: two threads read 7 and both write 8. A thread-safe collection doesn't make compound actions on it atomic (JCIP §4.4).
- **`merge(w, 1, Integer::sum)`:** the read-modify-write happens **inside** the map, atomically per key (`compute`, `computeIfAbsent` and `merge` all are).

---

## 5. Harness

### `RaceHarness.java (as built: in main sources at this stage)`

```java
package com.mastery.conc.race;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.function.Supplier;

/**
 * Reproduces each race: runs every implementation RUNS times with THREADS threads released
 * together by a latch, and reports how often (and how badly) the result was wrong.
 */
public final class RaceHarness {
    static final int RUNS = 20, THREADS = 8;

    /** Starts `threads` workers at the same instant and waits for all of them. */
    static void race(int threads, Runnable work) throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);
        List<Thread> ts = new ArrayList<>();
        for (int i = 0; i < threads; i++) {
            Thread t = new Thread(() -> {
                try { start.await(); } catch (InterruptedException e) { return; }
                work.run();
            });
            ts.add(t);
            t.start();
        }
        start.countDown();
        for (Thread t : ts) t.join();
    }

    static void counters() throws InterruptedException {
        int per = 250_000;
        long expected = (long) THREADS * per;
        List<Map.Entry<String, Supplier<HitCounter>>> impls = List.of(
                Map.entry("Racy (count++)", HitCounter.Racy::new),
                Map.entry("Synchronized", HitCounter.Synchronized::new),
                Map.entry("AtomicLong", HitCounter.Atomic::new),
                Map.entry("LongAdder", HitCounter.Adder::new));
        for (var impl : impls) {
            int wrong = 0;
            long worst = 0;
            for (int r = 0; r < RUNS; r++) {
                HitCounter c = impl.getValue().get();
                race(THREADS, () -> { for (int i = 0; i < per; i++) c.increment(); });
                long lost = expected - c.get();
                if (lost != 0) wrong++;
                worst = Math.max(worst, lost);
            }
            System.out.printf("HitCounter  %-16s wrong in %2d/%d runs; worst run lost %,d of %,d increments%n",
                    impl.getKey(), wrong, RUNS, worst, expected);
        }
    }

    static void seats() throws InterruptedException {
        int capacity = 1_000, attemptsPerThread = 1_000;
        List<Map.Entry<String, Supplier<SeatInventory>>> impls = List.of(
                Map.entry("Racy (check-act)", () -> new SeatInventory.Racy(capacity)),
                Map.entry("Synchronized", () -> new SeatInventory.Synchronized(capacity)),
                Map.entry("CAS loop", () -> new SeatInventory.Cas(capacity)));
        for (var impl : impls) {
            int wrong = 0, worstOversell = 0;
            for (int r = 0; r < RUNS; r++) {
                SeatInventory inv = impl.getValue().get();
                java.util.concurrent.atomic.AtomicInteger successes = new java.util.concurrent.atomic.AtomicInteger();
                race(THREADS, () -> { for (int i = 0; i < attemptsPerThread; i++) if (inv.tryBook()) successes.incrementAndGet(); });
                int oversold = successes.get() - capacity;           // tickets handed out beyond capacity
                if (oversold != 0 || inv.booked() != capacity) wrong++;
                worstOversell = Math.max(worstOversell, oversold);
            }
            System.out.printf("Seats       %-16s wrong in %2d/%d runs; worst run sold %,d tickets for %,d seats%n",
                    impl.getKey(), wrong, RUNS, capacity + worstOversell, capacity);
        }
    }

    static void words() throws InterruptedException {
        int per = 100_000;
        String[] vocabulary = new String[50];
        for (int i = 0; i < vocabulary.length; i++) vocabulary[i] = "w" + i;
        long expected = (long) THREADS * per;
        List<Map.Entry<String, Supplier<WordCounter>>> impls = List.of(
                Map.entry("HashMap", WordCounter.RacyHashMap::new),
                Map.entry("CHM get-then-put", WordCounter.GetThenPutOnConcurrentMap::new),
                Map.entry("CHM merge", WordCounter.Merge::new));
        for (var impl : impls) {
            int wrong = 0;
            long worst = 0;
            String failure = "";
            for (int r = 0; r < RUNS; r++) {
                WordCounter wc = impl.getValue().get();
                try {
                    race(THREADS, () -> { for (int i = 0; i < per; i++) wc.add(vocabulary[i % vocabulary.length]); });
                    long total = wc.snapshot().values().stream().mapToLong(Integer::longValue).sum();
                    if (total != expected) wrong++;
                    worst = Math.max(worst, expected - total);
                } catch (RuntimeException e) {
                    wrong++;
                    failure = " (also threw " + e.getClass().getSimpleName() + ")";
                }
            }
            System.out.printf("WordCounter %-16s wrong in %2d/%d runs; worst run lost %,d of %,d counts%s%n",
                    impl.getKey(), wrong, RUNS, worst, expected, failure);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println(THREADS + " threads, " + RUNS + " runs each, " + Runtime.getRuntime().availableProcessors() + " cores");
        counters();
        seats();
        words();
    }
}
```

The `CountDownLatch` releases all threads at the same instant, which maximises overlap. Starting threads one by one would let early threads finish before late ones start, hiding the race.

---

## 6. Contract Tests (as built)

### `ConcurrencyTestSupport.java`

```java
package com.mastery.conc.race;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;

/** Starts N threads at the same instant (latch) and waits for them; rethrows worker failures. */
final class ConcurrencyTestSupport {
    private ConcurrencyTestSupport() { }

    static void runConcurrently(int threads, Runnable work) throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);
        List<Thread> ts = new ArrayList<>();
        List<Throwable> failures = java.util.Collections.synchronizedList(new ArrayList<>());
        for (int i = 0; i < threads; i++) {
            Thread t = new Thread(() -> {
                try {
                    start.await();
                    work.run();
                } catch (Throwable e) {
                    failures.add(e);
                }
            });
            ts.add(t);
            t.start();
        }
        start.countDown();
        for (Thread t : ts) t.join();
        if (!failures.isEmpty()) {
            AssertionError e = new AssertionError(failures.size() + " worker(s) failed");
            failures.forEach(e::addSuppressed);
            throw e;
        }
    }
}
```

### `HitCounterContractTest.java (as built)`

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

    @Test
    void noLostUpdatesUnderContention() throws InterruptedException {
        HitCounter c = newCounter();
        int threads = 8, per = 100_000;
        ConcurrencyTestSupport.runConcurrently(threads, () -> { for (int i = 0; i < per; i++) c.increment(); });
        assertEquals((long) threads * per, c.get());
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

### `SeatInventoryContractTest.java (as built)`

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

    @Test
    void neverOversellsUnderContention() throws InterruptedException {
        int capacity = 1_000;
        SeatInventory inv = newInventory(capacity);
        AtomicInteger sold = new AtomicInteger();
        ConcurrencyTestSupport.runConcurrently(8, () -> {
            for (int i = 0; i < 1_000; i++) if (inv.tryBook()) sold.incrementAndGet();
        });
        assertEquals(capacity, sold.get(), "tickets handed out");
        assertEquals(capacity, inv.booked(), "seats recorded");
    }
}

class SynchronizedSeatInventoryTest extends SeatInventoryContractTest {
    @Override SeatInventory newInventory(int capacity) { return new SeatInventory.Synchronized(capacity); }
}

class CasSeatInventoryTest extends SeatInventoryContractTest {
    @Override SeatInventory newInventory(int capacity) { return new SeatInventory.Cas(capacity); }
}
```

### `WordCounterContractTest.java`

```java
package com.mastery.conc.race;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.Map;
import org.junit.jupiter.api.Test;

abstract class WordCounterContractTest {
    abstract WordCounter newCounter();

    @Test
    void countsWords() {
        WordCounter wc = newCounter();
        for (String w : "to be or not to be".split(" ")) wc.add(w);
        assertEquals(Map.of("to", 2, "be", 2, "or", 1, "not", 1), wc.snapshot());
    }

    @Test
    void noLostCountsUnderContention() throws InterruptedException {
        WordCounter wc = newCounter();
        int threads = 8, per = 50_000, vocabulary = 20;
        ConcurrencyTestSupport.runConcurrently(threads, () -> {
            for (int i = 0; i < per; i++) wc.add("w" + (i % vocabulary));
        });
        Map<String, Integer> snap = wc.snapshot();
        assertEquals(vocabulary, snap.size());
        for (int i = 0; i < vocabulary; i++) {
            assertEquals(threads * per / vocabulary, snap.get("w" + i), "count of w" + i);
        }
    }
}

class MergeWordCounterTest extends WordCounterContractTest {
    @Override WordCounter newCounter() { return new WordCounter.Merge(); }
}
```

```text
Tests run: 12 in com.mastery.conc.race, Failures: 0
(Synchronized/Atomic/Adder HitCounter, Synchronized/Cas SeatInventory, Merge WordCounter × 2 tests each)
```

`runConcurrently` collects worker failures and rethrows them, so an assertion failing *inside* a worker thread can't vanish silently (#042 §7).

> ⚠️ These tests pass on the fixes, but **do they fail on the racy versions?** That's not checked here. Row 50's review measures it, and the answer is "not reliably" (77% and 47% detection). See #050.

---

## 7. Which Fix to Ship (initial judgement; row 50 measures it)

| Race | Choice | Reason |
|------|--------|--------|
| Hit counter (metrics) | `LongAdder` | Write-heavy and shared; reads are rare and don't need to be atomic snapshots |
| Seats | `synchronized` | The check and the act must be atomic; a lock is simplest to reason about, and later extensions (seat maps, refunds) will need several fields under one lock |
| Word counts | `ConcurrentHashMap.merge` | Atomic per key, with concurrent access across different keys |

---

## Key Takeaways

1. **Reproduce first:** 8 threads + a latch + 20 runs turned "might be racy" into "wrong in 20/20 runs, lost 76%".
2. Two race shapes cover most bugs: **read-modify-write** (`count++`) and **check-then-act** (`if (x < cap) x++`). Both need the whole sequence to be atomic.
3. Races **compound**: lost increments hid the overselling (4.6× capacity).
4. A **thread-safe collection doesn't make compound actions atomic**; use `merge`/`compute`/`putIfAbsent`.
5. Fixes: a **lock** (any code, multiple variables), **CAS** (one variable, lock-free), **striping** (`LongAdder`, write-heavy counters).
6. Start concurrent tests with a **latch** and **rethrow worker failures**. And check that your tests can actually *fail* (row 50).
