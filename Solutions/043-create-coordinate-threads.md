# #043 — Solution: Create & Coordinate Threads

**Date:** 2026-10-12  
**Problem:** [043-create-coordinate-threads.md](../problems/043-create-coordinate-threads.md)

> Verified on JDK 21.0.10, Apple M4 (4 performance + 6 efficiency cores): **4/4 tests pass**, **zero `-Xlint:all` warnings**. All timings are measured.

---

## 1. Code

### `Primes.java`

```java
package com.mastery.conc.threads;

/** Deliberately CPU-bound work whose cost grows with n (trial division up to √n). */
public final class Primes {
    private Primes() { }

    public static boolean isPrime(long n) {
        if (n < 2) return false;
        if (n % 2 == 0) return n == 2;
        for (long d = 3; d * d <= n; d += 2) {
            if (n % d == 0) return false;
        }
        return true;
    }

    /** Sequential reference: number of primes in [from, to). */
    public static long countSequential(long from, long to) {
        long count = 0;
        for (long n = from; n < to; n++) {
            if (isPrime(n)) count++;
        }
        return count;
    }
}
```

### `ParallelPrimeCounter.java`

```java
package com.mastery.conc.threads;

/**
 * Counts primes in [0, limit) with N platform threads, using two partitioning strategies.
 *
 * <p>Coordination uses only Thread + join():
 * <ul>
 *   <li>each worker writes ONLY its own slot of {@code partial} (thread confinement: no shared
 *       mutable state, so no locks needed);</li>
 *   <li>{@code join()} gives happens-before: after it returns, the worker's write is visible.</li>
 * </ul>
 */
public final class ParallelPrimeCounter {
    private ParallelPrimeCounter() { }

    public enum Strategy {
        /** Thread i gets one contiguous block [i·limit/N, (i+1)·limit/N). */
        BLOCK,
        /** Thread i gets n = i, i+N, i+2N, ... (interleaved). */
        STRIPED,
        /** Threads repeatedly claim the next small chunk from a shared atomic counter (self-balancing). */
        DYNAMIC
    }

    /** Chunk size for DYNAMIC: small enough to balance, large enough that claiming is rare. */
    static final long CHUNK = 10_000;

    public static long count(long limit, int threads, Strategy strategy) throws InterruptedException {
        if (threads < 1) {
            throw new IllegalArgumentException("threads must be >= 1");
        }
        long[] partial = new long[threads];          // slot i is written only by worker i
        java.util.concurrent.atomic.AtomicLong next = new java.util.concurrent.atomic.AtomicLong();   // DYNAMIC only
        Thread[] workers = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            final int id = i;                        // lambdas capture effectively-final locals
            Runnable task = switch (strategy) {
                case BLOCK -> () -> {
                    long from = limit * id / threads;
                    long to = limit * (id + 1) / threads;
                    partial[id] = Primes.countSequential(from, to);
                };
                case STRIPED -> () -> {
                    long c = 0;                      // accumulate in a LOCAL (stack-confined), publish once
                    for (long n = id; n < limit; n += threads) {
                        if (Primes.isPrime(n)) c++;
                    }
                    partial[id] = c;
                };
                case DYNAMIC -> () -> {
                    long c = 0;
                    long start;
                    while ((start = next.getAndAdd(CHUNK)) < limit) {   // atomically claim [start, start+CHUNK)
                        c += Primes.countSequential(start, Math.min(start + CHUNK, limit));
                    }
                    partial[id] = c;
                };
            };
            workers[i] = new Thread(task, "prime-" + strategy + "-" + i);
            workers[i].start();
        }
        long total = 0;
        for (int i = 0; i < threads; i++) {
            workers[i].join();                       // wait + happens-before for partial[i]
            total += partial[i];
        }
        return total;
    }

    /** Per-thread busy time for the BLOCK strategy, to expose load imbalance. */
    public static long[] blockWorkerMillis(long limit, int threads) throws InterruptedException {
        long[] millis = new long[threads];
        Thread[] workers = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            final int id = i;
            workers[i] = new Thread(() -> {
                long t0 = System.nanoTime();
                Primes.countSequential(limit * id / threads, limit * (id + 1) / threads);
                millis[id] = (System.nanoTime() - t0) / 1_000_000;
            });
            workers[i].start();
        }
        for (Thread w : workers) {
            w.join();
        }
        return millis;
    }
}
```

### Why no lock is needed (Task 1.3)

- **Thread confinement:** slot `partial[i]` is written by exactly one thread (worker *i*), and nobody else writes it. There's no shared *mutable* state between workers, and so no race. The running count `c` is a **local** variable, on the worker's own stack (#004).
- **`join()` publishes the result:** the JMM guarantees that *all actions of a thread happen-before another thread returns from `join()` on it* (#012 §4). So after `workers[i].join()`, the coordinator is guaranteed to see `partial[i]`. Reading `partial[i]` **without** joining would be a data race (it could read 0).
- `limit` and `threads` are captured **effectively final** locals: immutable, so safe to share.
- The `AtomicLong next` (DYNAMIC only) *is* shared and mutable. `getAndAdd` makes "claim the next chunk" one atomic step. With a plain `long`, two threads could claim the same chunk (row 49 is all about that).

### `CancellableSearch.java`

```java
package com.mastery.conc.threads;

import java.util.OptionalLong;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.LongPredicate;

/**
 * Searches [from, to) with N threads for ANY number matching a predicate. As soon as one worker
 * finds a match, the coordinator INTERRUPTS the others (cooperative cancellation).
 */
public final class CancellableSearch {
    private CancellableSearch() { }

    private static final long NONE = Long.MIN_VALUE;

    public static OptionalLong findAny(long from, long to, int threads, LongPredicate predicate)
            throws InterruptedException {
        AtomicLong found = new AtomicLong(NONE);     // written by whichever worker wins (row 49 explains Atomic*)
        Thread[] workers = new Thread[threads];
        Thread coordinator = Thread.currentThread();
        for (int i = 0; i < threads; i++) {
            final int id = i;
            workers[i] = new Thread(() -> {
                for (long n = from + id; n < to; n += threads) {
                    if (Thread.currentThread().isInterrupted()) {
                        return;                       // cooperative: we check the flag ourselves
                    }
                    if (predicate.test(n)) {
                        if (found.compareAndSet(NONE, n)) {
                            coordinator.interrupt();  // wake the coordinator: we have an answer
                        }
                        return;
                    }
                }
            }, "search-" + i);
            workers[i].start();
        }
        try {
            for (Thread w : workers) {
                w.join();                             // returns early (InterruptedException) if a worker wins
            }
        } catch (InterruptedException wokenByWinner) {
            if (found.get() == NONE) {
                throw wokenByWinner;                  // a real external interrupt: propagate it
            }
        } finally {
            for (Thread w : workers) {
                w.interrupt();                        // stop everyone still searching
            }
            for (Thread w : workers) {
                w.join();                             // and wait until they have actually stopped
            }
        }
        long result = found.get();
        return result == NONE ? OptionalLong.empty() : OptionalLong.of(result);
    }
}
```

### How cancellation works (Task 3)

1. Each worker polls `isInterrupted()` once per number; that's the **cooperative** part (#042 §6).
2. The first worker to find a match wins the `compareAndSet(NONE, n)`; only the winner interrupts the coordinator. A second finder loses the CAS and just returns, so there's exactly one wake-up.
3. The coordinator is blocked in `join()`; the interrupt makes it throw `InterruptedException`, which **also clears its flag**. That's why the test can assert the flag is clean.
4. `finally` interrupts every worker, then **joins them all**, so no stray thread outlives the call.
5. If the coordinator was interrupted but nobody found anything, the interrupt came from **outside**, so it's rethrown, not swallowed.

### `WorkerFailures.java`

```java
package com.mastery.conc.threads;

import java.util.List;
import java.util.concurrent.ConcurrentLinkedQueue;

/**
 * Exceptions in a thread don't reach the thread that started it. This runs tasks on threads,
 * collects their failures with an UncaughtExceptionHandler, and rethrows in the caller.
 */
public final class WorkerFailures {
    private WorkerFailures() { }

    public static void runAll(List<Runnable> tasks) throws InterruptedException {
        ConcurrentLinkedQueue<Throwable> failures = new ConcurrentLinkedQueue<>();   // thread-safe collection
        Thread[] threads = new Thread[tasks.size()];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(tasks.get(i), "task-" + i);
            threads[i].setUncaughtExceptionHandler((t, e) -> failures.add(e));
            threads[i].start();
        }
        for (Thread t : threads) {
            t.join();                                 // the handler runs before the thread terminates → visible after join
        }
        if (!failures.isEmpty()) {
            RuntimeException combined = new RuntimeException(failures.size() + " task(s) failed");
            failures.forEach(combined::addSuppressed);
            throw combined;
        }
    }
}
```

The uncaught-exception handler runs **on the dying thread, before it terminates**, so `join()` makes its write to `failures` visible to the caller. `ConcurrentLinkedQueue` because several handlers may add at the same moment. (Executors give you this for free via `Future.get()` → `ExecutionException`, as verified in #042.)

---

## 2. Tests

### `ThreadsTest.java`

```java
package com.mastery.conc.threads;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;
import static org.junit.jupiter.api.Assertions.assertTrue;

import com.mastery.conc.threads.ParallelPrimeCounter.Strategy;
import java.time.Duration;
import java.util.List;
import java.util.OptionalLong;
import org.junit.jupiter.api.Test;

class ThreadsTest {

    @Test
    void bothStrategiesMatchSequentialForManyThreadCounts() throws InterruptedException {
        long limit = 200_000;
        long expected = Primes.countSequential(0, limit);
        assertEquals(17_984, expected);                               // π(200 000) = 17 984
        for (int threads : new int[]{1, 2, 3, 7, 10, 16}) {
            assertEquals(expected, ParallelPrimeCounter.count(limit, threads, Strategy.BLOCK), "BLOCK " + threads);
            assertEquals(expected, ParallelPrimeCounter.count(limit, threads, Strategy.STRIPED), "STRIPED " + threads);
            assertEquals(expected, ParallelPrimeCounter.count(limit, threads, Strategy.DYNAMIC), "DYNAMIC " + threads);
        }
    }

    @Test
    void searchFindsAMatchAndStopsTheOtherWorkers() {
        // The only multiples of 1_000_003 in range are far from the start; workers must cooperate to stop.
        OptionalLong r = assertTimeoutPreemptively(Duration.ofSeconds(5), () ->
                CancellableSearch.findAny(1, Long.MAX_VALUE, 4, n -> n % 1_000_003 == 0));
        assertTrue(r.isPresent());
        assertEquals(0, r.getAsLong() % 1_000_003);
        assertFalse(Thread.currentThread().isInterrupted(), "coordinator must not leak an interrupt");
    }

    @Test
    void searchReturnsEmptyWhenNothingMatches() throws InterruptedException {
        assertEquals(OptionalLong.empty(), CancellableSearch.findAny(0, 10_000, 3, n -> n < 0));
    }

    @Test
    void workerExceptionsAreSurfacedToTheCaller() {
        RuntimeException e = assertThrows(RuntimeException.class, () -> WorkerFailures.runAll(List.of(
                () -> { },
                () -> { throw new IllegalStateException("disk full"); },
                () -> { throw new IllegalArgumentException("bad input"); })));
        assertEquals("2 task(s) failed", e.getMessage());
        assertEquals(2, e.getSuppressed().length);
    }
}
```

```text
$ mvn test -Dtest=ThreadsTest
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
```

---

## 3. Measured Speedup (Task 2)

### `ScalingDemo.java`

```java
package com.mastery.conc.threads;

import com.mastery.conc.threads.ParallelPrimeCounter.Strategy;
import java.util.Arrays;

/** Wall-clock speedup of the parallel prime count (best of 3 runs after a warm-up). */
public final class ScalingDemo {
    public static void main(String[] args) throws InterruptedException {
        long limit = 10_000_000;
        ParallelPrimeCounter.count(limit / 10, 4, Strategy.STRIPED);   // warm-up (JIT)
        long base = best(limit, 1, Strategy.BLOCK);
        System.out.printf("limit=%,d  primes=%,d  available processors=%d%n", limit,
                ParallelPrimeCounter.count(limit, 4, Strategy.STRIPED), Runtime.getRuntime().availableProcessors());
        System.out.printf("%-8s %10s %8s %11s %8s %11s %8s%n", "threads", "BLOCK ms", "speedup", "STRIPED ms", "speedup", "DYNAMIC ms", "speedup");
        for (int t : new int[]{1, 2, 4, 6, 8, 10, 16}) {
            long b = best(limit, t, Strategy.BLOCK);
            long s = best(limit, t, Strategy.STRIPED);
            long d = best(limit, t, Strategy.DYNAMIC);
            System.out.printf("%-8d %10d %7.2fx %11d %7.2fx %11d %7.2fx%n", t, b, (double) base / b, s, (double) base / s, d, (double) base / d);
        }
        System.out.println("BLOCK, 4 threads, per-thread busy ms: " + Arrays.toString(ParallelPrimeCounter.blockWorkerMillis(limit, 4)));
    }

    static long best(long limit, int threads, Strategy s) throws InterruptedException {
        long best = Long.MAX_VALUE;
        for (int i = 0; i < 3; i++) {
            long t0 = System.nanoTime();
            ParallelPrimeCounter.count(limit, threads, s);
            best = Math.min(best, (System.nanoTime() - t0) / 1_000_000);
        }
        return best;
    }
}
```

```text
limit=10,000,000  primes=664,579  available processors=10
threads    BLOCK ms  speedup  STRIPED ms  speedup  DYNAMIC ms  speedup
1               567    1.00x         571    0.99x         569    0.99x
2               356    1.59x         578    0.98x         289    1.96x
4               192    2.95x         294    1.93x         149    3.80x
6               161    3.52x         290    1.95x         133    4.26x
8               146    3.88x         149    3.80x         123    4.60x
10              124    4.56x         148    3.82x         115    4.92x
16              121    4.68x         131    4.32x         116    4.88x
BLOCK, 4 threads, per-thread busy ms: [87, 144, 171, 194]
```

### Explaining the results

**BLOCK: load imbalance.** `isPrime(n)` costs up to ~√n divisions, so later blocks hold more expensive numbers. With 4 threads the busy times were **87 / 144 / 171 / 194 ms**: the last thread works 2.2× as long as the first, and the wall time is set by the slowest. The speedup at 2 threads is only 1.59×.

**STRIPED: the surprise.** I predicted striping would balance well, since every thread gets a mix of small and large numbers. It was **worse** than BLOCK at 2, 4 and 6 threads:

| Threads | Thread *i* gets numbers ≡ *i* (mod N) | Threads doing real work | Speedup |
|--------:|----------------------------------------|------------------------:|--------:|
| 2 | Thread 0: **all even numbers** (rejected instantly by `n % 2`); thread 1: all the odd ones | **1** | **0.98×** |
| 4 | ids 0, 2 even → trivial; ids 1, 3 odd | 2 | 1.93× |
| 6 | ids 0, 2, 4 even; id 3 ≡ 3 (mod 6), divisible by 3 → rejected at the first divisor; only ids 1, 5 do real work | 2 | 1.95× |
| 8 | ids 1, 3, 5, 7 | 4 | 3.80× |
| 10 | ids 1, 3, 7, 9 (id 5: multiples of 5 → rejected early) | 4 | 3.82× |

The stride **resonates with the structure of the data**: the work isn't uniform across residues mod N. Striping only balances work that's statistically uniform across positions.

**DYNAMIC: the fix.** Every thread repeatedly claims the next 10 000-number chunk from a shared `AtomicLong` (`getAndAdd`). Fast threads simply claim more chunks, so the load balances **whatever the data looks like**. It's the best at every thread count: **1.96× at 2, 3.80× at 4** (near-linear), 4.92× at 10. The cost is one atomic operation per 10 000 numbers, which is negligible. This is the idea behind **work queues**, thread pools and `ForkJoinPool` work-stealing (rows 67+).

**Why speedup stops near 5× with 10 threads:** the M4 has **4 performance cores + 6 efficiency cores**. Up to 4 threads, scaling is near-linear (3.80×). The 6 E-cores add less than one P-core's worth of throughput each, and 16 threads (more than the cores) only adds context-switching. **Amdahl's law** also applies: the sequential parts (thread creation, the final sum) cap the speedup.

---

## Key Takeaways

1. **Thread confinement + `join()`** is the simplest correct coordination: each thread owns its output, and `join` publishes it.
2. **How you partition decides how fast you go.** Static partitions (BLOCK, STRIPED) balance only if the work is uniform, and the data can defeat them in surprising ways (stride resonance).
3. **Dynamic scheduling** (claim small chunks from a shared atomic counter) balances automatically. It's what thread pools and work-stealing generalise.
4. Speedup is bounded by **real cores** (and their types), plus Amdahl's sequential fraction. More threads than cores doesn't help CPU-bound work.
5. **Cancellation is cooperative:** poll the interrupt flag; the winner wakes the coordinator; `finally` stops and joins everyone; don't leak or swallow interrupts.
6. Worker exceptions don't propagate by themselves. Collect them (handler, or later `Future.get()`) and rethrow in the caller.
