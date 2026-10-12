# #047 — Solution: Solve 2–3 Exercises on Concurrency

**Date:** 2026-10-12  
**Problem:** [047-concurrency-exercises.md](../problems/047-concurrency-exercises.md)

> Verified on JDK 21.0.10, Apple M4: **44/44 test executions pass** (two tests repeated 20× each, plus 4 more), **zero `-Xlint:all` warnings**.

---

## The Common Pattern

All three exercises are the same shape (JCIP §14.1, "condition predicates"):

```java
synchronized (lock) {
    while (!myConditionHolds()) {   // 1. wait for MY condition (in a loop)
        lock.wait();
    }
    doMyStep();                     // 2. act while holding the lock
    changeSharedState();            // 3. update the state others are waiting on
    lock.notifyAll();               // 4. tell everyone to re-check
}
```

For each exercise, the only design question is **what the shared state is** and **what each thread's predicate is**.

| Exercise | Shared state (guarded by `lock`) | Thread's predicate |
|----------|----------------------------------|--------------------|
| Ping-pong | `boolean pingsTurn` | "it's my turn": `pingsTurn == isPing` |
| Round-robin | `int next` | "`next` is mine": `(next − 1) % K == id`, **or** done (`next > max`) |
| Latch | `long count` | "the gate is open": `count == 0` |

---

## Exercise 1 — Ping-Pong

### `PingPong.java`

```java
package com.mastery.conc.exercises;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Exercise 1: two threads print "ping" and "pong" strictly alternately, N times each.
 * Shared state: whose turn it is, guarded by one lock; each thread waits for its turn.
 */
public final class PingPong {
    private final Object lock = new Object();
    private boolean pingsTurn = true;                       // guarded by lock
    private final List<String> log = Collections.synchronizedList(new ArrayList<>());

    private void play(boolean isPing, int rounds) throws InterruptedException {
        for (int i = 0; i < rounds; i++) {
            synchronized (lock) {
                while (pingsTurn != isPing) {               // wait for MY turn (condition predicate)
                    lock.wait();
                }
                log.add(isPing ? "ping" : "pong");
                pingsTurn = !isPing;                        // hand the turn over
                lock.notifyAll();
            }
        }
    }

    /** Runs both players and returns the observed sequence. */
    public static List<String> run(int rounds) throws InterruptedException {
        PingPong game = new PingPong();
        Thread ping = new Thread(() -> { try { game.play(true, rounds); } catch (InterruptedException e) { Thread.currentThread().interrupt(); } }, "ping");
        Thread pong = new Thread(() -> { try { game.play(false, rounds); } catch (InterruptedException e) { Thread.currentThread().interrupt(); } }, "pong");
        pong.start();                                       // start pong FIRST: it must still wait for ping
        ping.start();
        ping.join();
        pong.join();
        return List.copyOf(game.log);
    }
}
```

- **Starting `pong` first** is the tricky case: it enters, finds `pingsTurn == true != isPing`, and waits. The order of `start()` calls doesn't matter, because the **state** decides who goes.
- `notifyAll` (not `notify`) would be needed as soon as there are more than two players. With exactly two it would happen to work, but `notifyAll` keeps the code correct if someone adds a third.
- The log is a `synchronizedList`, but every `add` happens inside `synchronized (lock)` anyway. The extra wrapper just makes the final copy safe without reasoning.

---

## Exercise 2 — Round-Robin Printer

### `RoundRobinPrinter.java`

```java
package com.mastery.conc.exercises;

import java.util.ArrayList;
import java.util.List;

/**
 * Exercise 2: K threads print 1..max in order; thread i prints exactly the numbers n with
 * (n - 1) % K == i. Generalises PingPong from 2 turns to K turns.
 */
public final class RoundRobinPrinter {
    private final Object lock = new Object();
    private final int threads;
    private final int max;
    private int next = 1;                                    // guarded by lock: the next number to print
    private final List<String> out = new ArrayList<>();      // guarded by lock

    private RoundRobinPrinter(int threads, int max) {
        this.threads = threads;
        this.max = max;
    }

    private void worker(int id) throws InterruptedException {
        synchronized (lock) {
            while (true) {
                while (next <= max && (next - 1) % threads != id) {   // not my number yet
                    lock.wait();
                }
                if (next > max) {
                    lock.notifyAll();                       // let the others observe "done" too
                    return;
                }
                out.add("T" + id + ":" + next);
                next++;
                lock.notifyAll();
            }
        }
    }

    public static List<String> run(int threads, int max) throws InterruptedException {
        RoundRobinPrinter p = new RoundRobinPrinter(threads, max);
        List<Thread> ts = new ArrayList<>();
        for (int i = threads - 1; i >= 0; i--) {             // start in REVERSE order on purpose
            final int id = i;
            Thread t = new Thread(() -> { try { p.worker(id); } catch (InterruptedException e) { Thread.currentThread().interrupt(); } }, "rr-" + id);
            ts.add(t);
            t.start();
        }
        for (Thread t : ts) t.join();
        synchronized (p.lock) {
            return List.copyOf(p.out);
        }
    }
}
```

- **The termination trap:** a naive predicate `while ((next − 1) % K != id) wait();` never becomes true for threads whose next number would exceed `max`, so they wait forever and `join()` hangs. The fix is to make "done" part of the predicate (`next <= max && …`), and to `notifyAll` on exit so the others notice too. The test `K = 8, max = 3` is there for exactly this case.
- Starting threads in **reverse** order proves the output depends on the shared state, not on start order or scheduling.
- Each print triggers `notifyAll`, which wakes **all** K threads to re-check, and K−1 of them go back to sleep. That's fine for K = 4. For large K, `Condition` objects per thread (rows 57+) avoid the thundering herd.

---

## Exercise 3 — `SimpleCountDownLatch`

### `SimpleCountDownLatch.java`

```java
package com.mastery.conc.exercises;

import java.util.concurrent.TimeUnit;

/**
 * Exercise 3: a minimal CountDownLatch built on an intrinsic lock.
 *
 * <p>Invariant: {@code count >= 0}; once it reaches 0 it stays 0 (a latch is one-shot).
 * Happens-before (like the JDK's): actions before {@code countDown()} happen-before a successful
 * return from {@code await()}, because both go through the same monitor.
 */
public final class SimpleCountDownLatch {
    private final Object lock = new Object();
    private long count;                                      // guarded by lock

    public SimpleCountDownLatch(long count) {
        if (count < 0) {
            throw new IllegalArgumentException("count < 0");
        }
        this.count = count;
    }

    public void countDown() {
        synchronized (lock) {
            if (count > 0 && --count == 0) {
                lock.notifyAll();                            // open the gate for EVERY waiter
            }
        }
    }

    public void await() throws InterruptedException {
        synchronized (lock) {
            while (count > 0) {
                lock.wait();
            }
        }
    }

    /** Waits at most the given time; returns true if the count reached zero. */
    public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        synchronized (lock) {
            while (count > 0) {
                long remainingNanos = deadline - System.nanoTime();
                if (remainingNanos <= 0) {
                    return false;
                }
                // wait() takes millis (+ nanos); round UP so we never spin with wait(0) = wait forever
                long millis = Math.max(1, TimeUnit.NANOSECONDS.toMillis(remainingNanos));
                lock.wait(millis);                           // may return early: spurious, notify, or timeout
            }
            return true;
        }
    }

    public long getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

- **Why `notifyAll`:** a latch must release **every** waiter at once, not one.
- **Happens-before:** the writer's `data[0] = 42` happens-before its `countDown()` (program order). `countDown()` **unlocks** the monitor, and the waiter's `await()` later **locks** the same monitor and sees `count == 0`. Unlock → lock of the same monitor is a happens-before edge (#045 §5, #048 §7), and by transitivity the waiter sees 42. The test checks this 1 000 times with a plain, non-volatile field. The JDK's `CountDownLatch` documents the same guarantee ("actions prior to calling countDown() happen-before actions following a successful return from await()").
- **Timed `await`:** `wait(ms)` can return early (a notification, a spurious wake-up), so each loop iteration recomputes `remaining = deadline − now` against a fixed `deadline` taken once at the start. Two subtleties: `Math.max(1, …)`, because **`wait(0)` means "wait forever"**, and `System.nanoTime()` (monotonic) rather than `currentTimeMillis()` (which can jump if the wall clock is adjusted).
- **One-shot:** the `count > 0 &&` guard keeps the count at 0 under extra `countDown()` calls.

---

## Tests

### `ExercisesTest.java`

```java
package com.mastery.conc.exercises;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.Test;

class ExercisesTest {

    @RepeatedTest(20)
    void pingPongAlternatesStrictly() {
        List<String> log = assertTimeoutPreemptively(Duration.ofSeconds(5), () -> PingPong.run(1_000));
        assertEquals(2_000, log.size());
        for (int i = 0; i < log.size(); i++) {
            assertEquals(i % 2 == 0 ? "ping" : "pong", log.get(i), "position " + i);
        }
    }

    @RepeatedTest(20)
    void roundRobinPrintsInOrderWithTheRightThread() {
        int threads = 4, max = 500;
        List<String> out = assertTimeoutPreemptively(Duration.ofSeconds(5), () -> RoundRobinPrinter.run(threads, max));
        assertEquals(max, out.size());
        for (int n = 1; n <= max; n++) {
            assertEquals("T" + ((n - 1) % threads) + ":" + n, out.get(n - 1));
        }
    }

    @Test
    void roundRobinWithMoreThreadsThanNumbersTerminates() {
        List<String> out = assertTimeoutPreemptively(Duration.ofSeconds(5), () -> RoundRobinPrinter.run(8, 3));
        assertEquals(List.of("T0:1", "T1:2", "T2:3"), out);
    }

    @Test
    void latchReleasesAllWaitersOnlyAtZero() {
        assertTimeoutPreemptively(Duration.ofSeconds(5), () -> {
            SimpleCountDownLatch latch = new SimpleCountDownLatch(3);
            AtomicInteger released = new AtomicInteger();
            List<Thread> waiters = new ArrayList<>();
            for (int i = 0; i < 5; i++) {
                Thread t = new Thread(() -> {
                    try { latch.await(); released.incrementAndGet(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                });
                waiters.add(t);
                t.start();
            }
            for (Thread t : waiters) {                       // all 5 must be parked in wait() before we count down
                while (t.getState() != Thread.State.WAITING) Thread.onSpinWait();
            }
            latch.countDown();
            latch.countDown();
            assertEquals(0, released.get(), "must not release before zero");
            latch.countDown();
            for (Thread t : waiters) t.join();
            assertEquals(5, released.get());
            latch.countDown();                               // extra countDown: stays at 0
            assertEquals(0, latch.getCount());
        });
    }

    @Test
    void latchPublishesWritesMadeBeforeCountDown() {
        assertTimeoutPreemptively(Duration.ofSeconds(5), () -> {
            for (int round = 0; round < 1_000; round++) {
                SimpleCountDownLatch latch = new SimpleCountDownLatch(1);
                int[] data = new int[1];                      // plain, non-volatile
                Thread writer = new Thread(() -> { data[0] = 42; latch.countDown(); });
                writer.start();
                latch.await();
                assertEquals(42, data[0], "happens-before via the latch");
                writer.join();
            }
        });
    }

    @Test
    void timedAwaitTimesOutAndSucceeds() throws InterruptedException {
        SimpleCountDownLatch latch = new SimpleCountDownLatch(1);
        long t0 = System.nanoTime();
        assertFalse(latch.await(50, TimeUnit.MILLISECONDS));
        long waitedMs = (System.nanoTime() - t0) / 1_000_000;
        assertTrue(waitedMs >= 50 && waitedMs < 1_000, "waited " + waitedMs + " ms");
        latch.countDown();
        assertTrue(latch.await(0, TimeUnit.MILLISECONDS) || latch.await(1, TimeUnit.MILLISECONDS));
        assertThrows(IllegalArgumentException.class, () -> new SimpleCountDownLatch(-1));
    }
}
```

```text
$ mvn test -Dtest=ExercisesTest
Tests run: 44, Failures: 0, Errors: 0, Skipped: 0
```

- **`@RepeatedTest(20)`** on the two ordering tests: a lesson from #050, where a single run of a concurrency test caught bugs only 47–77% of the time.
- The latch test waits until all 5 waiters are **actually in `WAITING`** before counting down. Otherwise "they weren't released early" could pass simply because they hadn't started waiting yet.

---

## Key Takeaways

1. Every wait/notify problem reduces to: **shared state + a condition predicate per thread + `while (!pred) wait()` + `notifyAll` after changing state**.
2. **Termination** must be part of the predicate, or some threads wait forever.
3. Ordering comes from **state**, not from start order or timing.
4. A synchronizer built on one monitor automatically provides **happens-before** from release to acquire.
5. Timed waits: a **fixed deadline**, recompute the remaining time each loop, never `wait(0)` by accident, and use `nanoTime`.
6. **Repeat** concurrency tests, and make sure the precondition (e.g. "threads are waiting") is really established before asserting.
