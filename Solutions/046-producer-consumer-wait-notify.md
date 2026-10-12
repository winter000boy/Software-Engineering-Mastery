# #046 — Solution: Producer-Consumer with `wait`/`notify`

**Date:** 2026-10-12  
**Problem:** [046-producer-consumer-wait-notify.md](../problems/046-producer-consumer-wait-notify.md)

> Verified on JDK 21.0.10, Apple M4: **6/6 tests pass**, **zero `-Xlint:all` warnings**, and the broken-variant harness results below were measured (50 runs each).

---

## 1. `BoundedBuffer<E>`

### `BoundedBuffer.java`

```java
package com.mastery.conc.pc;

/**
 * A fixed-capacity FIFO buffer for producer-consumer hand-off, built only on an intrinsic lock
 * and its wait set (the "Java monitor pattern", JCIP §4.2.1).
 *
 * <p>Invariant (guarded by {@code lock}): {@code 0 <= count <= items.length}; the elements are
 * {@code items[head], items[head+1], ...} (circular), {@code count} of them.
 *
 * <p>Condition predicates:
 * <ul>
 *   <li>put may proceed when {@code count < items.length} ("not full")</li>
 *   <li>take may proceed when {@code count > 0} ("not empty")</li>
 * </ul>
 */
public final class BoundedBuffer<E> {
    private final Object lock = new Object();   // private: no outside code can wait/notify on it
    private final Object[] items;               // circular array, like ArrayDeque/ArrayBlockingQueue
    private int head;                           // index of the next element to take
    private int count;

    public BoundedBuffer(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be positive: " + capacity);
        }
        this.items = new Object[capacity];
    }

    /** Blocks while the buffer is full. */
    public void put(E e) throws InterruptedException {
        java.util.Objects.requireNonNull(e, "element");
        synchronized (lock) {
            while (count == items.length) {     // WHILE, not if: re-check after every wake-up
                lock.wait();                    // releases `lock`, waits, re-acquires before returning
            }
            items[(head + count) % items.length] = e;
            count++;
            lock.notifyAll();                   // state changed: wake waiters to re-check their predicates
        }
    }

    /** Blocks while the buffer is empty. */
    public E take() throws InterruptedException {
        synchronized (lock) {
            while (count == 0) {
                lock.wait();
            }
            @SuppressWarnings("unchecked")      // safe: only E's are ever stored by put
            E e = (E) items[head];
            items[head] = null;                 // let GC reclaim it (EJ Item 7)
            head = (head + 1) % items.length;
            count--;
            lock.notifyAll();
            return e;
        }
    }

    public int size() {
        synchronized (lock) {                   // even reads need the lock (visibility, #045 §5)
            return count;
        }
    }

    public int capacity() {
        return items.length;                    // immutable: no lock needed
    }
}
```

### Design notes

- **A circular array** (`head` + `count`; the tail is `(head + count) % capacity`): O(1) `put`/`take` with no shifting. It's the same layout as `ArrayDeque` and `ArrayBlockingQueue`.
- **Why a private lock matters *especially* for `wait`/`notify`:** if we locked `this`, any outside code holding a reference could call `buffer.wait()` (and be woken by *our* `notifyAll`) or `buffer.notify()` (and consume a wake-up meant for our producers or consumers). A private lock means only this class can touch the wait set.
- **`wait()` releases the lock while waiting** (otherwise nobody could ever make room) and **re-acquires it before returning**, so the code after `wait()` is still inside the critical section.
- **`size()` takes the lock too**: reading `count` without it could see a stale value (#045 §5).
- **`capacity()` doesn't:** `items` is a `final` field and its length never changes.

### Why `while`, precisely

Waking up from `wait()` doesn't mean the predicate is now true:

1. **Another thread got there first.** `notifyAll` wakes *all* waiters, but they re-acquire the lock one at a time. The first consumer takes the only item; the second consumer, waking next, finds the buffer **empty again**.
2. **Spurious wake-ups:** the JLS (§17.2.1) explicitly allows `wait()` to return without any notify.
3. **One wait set, many predicates:** producers wait for "not full" and consumers for "not empty", on the same lock. A wake-up may be "for" the other kind of thread.

So: **always** `while (!predicate) lock.wait();`.

### Why `notifyAll`

Producers and consumers wait on the **same** wait set for **different** predicates. `notify()` wakes **one arbitrary** waiter, possibly one of the *wrong kind*, which re-checks its predicate, finds it false, and goes back to sleep **without passing the signal on**. The signal is lost, and eventually everyone waits (measured in §3). `notifyAll` wakes everyone; each re-checks its own predicate. The cost is extra wake-ups, which is negligible here. (JCIP §14.2.4: use `notify` only when all waiters wait for the *same* predicate *and* each notification lets at most one waiter proceed. Explicit `Condition` objects, rows 57+, let producers and consumers wait in **separate** wait sets.)

---

## 2. Tests

### `BoundedBufferTest.java`

```java
package com.mastery.conc.pc;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTimeoutPreemptively;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicReference;
import org.junit.jupiter.api.Test;

class BoundedBufferTest {

    /** Waits (bounded) until a thread reaches the given state; avoids sleep-based guessing. */
    private static void awaitState(Thread t, Thread.State state) throws InterruptedException {
        long deadline = System.nanoTime() + 2_000_000_000L;
        while (t.getState() != state) {
            if (System.nanoTime() > deadline) {
                throw new AssertionError(t.getName() + " never reached " + state + ", is " + t.getState());
            }
            Thread.onSpinWait();
        }
    }

    @Test
    void fifoOrderSingleThread() throws InterruptedException {
        BoundedBuffer<String> b = new BoundedBuffer<>(3);
        b.put("a");
        b.put("b");
        b.put("c");
        assertEquals("a", b.take());
        b.put("d");                                   // wraps around the circular array
        assertEquals(List.of("b", "c", "d"), List.of(b.take(), b.take(), b.take()));
        assertEquals(0, b.size());
    }

    @Test
    void putBlocksWhenFullUntilATakeMakesRoom() throws InterruptedException {
        BoundedBuffer<Integer> b = new BoundedBuffer<>(1);
        b.put(1);
        Thread producer = new Thread(() -> {
            try { b.put(2); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "blocked-producer");
        producer.start();
        awaitState(producer, Thread.State.WAITING);   // inside lock.wait(): blocked by "full"
        assertEquals(1, b.take());
        producer.join(2_000);
        assertEquals(Thread.State.TERMINATED, producer.getState());
        assertEquals(2, b.take());
    }

    @Test
    void takeBlocksWhenEmptyAndIsInterruptible() throws InterruptedException {
        BoundedBuffer<Integer> b = new BoundedBuffer<>(2);
        AtomicReference<Throwable> outcome = new AtomicReference<>();
        Thread consumer = new Thread(() -> {
            try { b.take(); } catch (Throwable t) { outcome.set(t); }
        }, "blocked-consumer");
        consumer.start();
        awaitState(consumer, Thread.State.WAITING);
        consumer.interrupt();                         // cancellation must work while waiting
        consumer.join(2_000);
        assertTrue(outcome.get() instanceof InterruptedException);
    }

    @Test
    void manyProducersAndConsumersDeliverEveryItemExactlyOnce() {
        assertTimeoutPreemptively(Duration.ofSeconds(20), () -> {
            int producers = 6, consumers = 4, perProducer = 50_000;
            BoundedBuffer<Long> b = new BoundedBuffer<>(8);           // small capacity: lots of blocking
            ConcurrentHashMap<Long, Boolean> seen = new ConcurrentHashMap<>();
            AtomicReference<String> duplicate = new AtomicReference<>();
            List<Thread> threads = new ArrayList<>();
            for (int p = 0; p < producers; p++) {
                final long base = (long) p * perProducer;            // unique ids per producer
                threads.add(new Thread(() -> {
                    try { for (int i = 0; i < perProducer; i++) b.put(base + i); }
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }));
            }
            int perConsumer = producers * perProducer / consumers;
            for (int c = 0; c < consumers; c++) {
                threads.add(new Thread(() -> {
                    try {
                        for (int i = 0; i < perConsumer; i++) {
                            Long v = b.take();
                            if (seen.putIfAbsent(v, Boolean.TRUE) != null) duplicate.set("duplicate " + v);
                        }
                    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }));
            }
            threads.forEach(Thread::start);
            for (Thread t : threads) t.join();
            assertEquals(null, duplicate.get());
            assertEquals(producers * perProducer, seen.size());       // nothing lost
            assertEquals(0, b.size());
        });
    }

    @Test
    void perProducerOrderIsPreserved() {
        assertTimeoutPreemptively(Duration.ofSeconds(10), () -> {
            BoundedBuffer<int[]> b = new BoundedBuffer<>(4);
            int producers = 3, perProducer = 20_000;
            for (int p = 0; p < producers; p++) {
                final int id = p;
                new Thread(() -> {
                    try { for (int i = 0; i < perProducer; i++) b.put(new int[]{id, i}); }
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }).start();
            }
            int[] lastSeq = {-1, -1, -1};
            for (int i = 0; i < producers * perProducer; i++) {      // single consumer: FIFO ⇒ per-producer order
                int[] item = b.take();
                assertTrue(item[1] > lastSeq[item[0]], "out of order for producer " + item[0]);
                lastSeq[item[0]] = item[1];
            }
        });
    }

    @Test
    void rejectsBadArguments() {
        assertThrows(IllegalArgumentException.class, () -> new BoundedBuffer<String>(0));
        assertThrows(NullPointerException.class, () -> new BoundedBuffer<String>(1).put(null));
    }
}
```

```text
$ mvn test -Dtest=BoundedBufferTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
```

**Testing blocking without `sleep`:** `awaitState` spins (with `Thread.onSpinWait()`, which is cheap) until the producer is actually in the `WAITING` state, meaning it's inside `lock.wait()`. That's a **fact** about the thread, not a guess about timing. Every multi-threaded test is wrapped in `assertTimeoutPreemptively`, so a deadlock bug **fails** the test in seconds instead of hanging the build forever.

**Exactly-once check:** each producer emits unique ids (`p × perProducer + i`); consumers record what they receive in a `ConcurrentHashMap`. Any duplicate is flagged, and `seen.size()` == total proves nothing was lost.

---

## 3. Breaking It on Purpose (measured)

### `BrokenBuffers.java`

```java
package com.mastery.conc.pc;

/**
 * Two classic wait/notify mistakes, kept ONLY to demonstrate how they fail. Do not use.
 */
public final class BrokenBuffers {
    private BrokenBuffers() { }

    /** Common interface so the same harness can drive every variant. */
    public interface Buffer {
        void put(int e) throws InterruptedException;
        int take() throws InterruptedException;
    }

    /** Mistake 1: `if` instead of `while` around wait(). */
    public static final class IfInsteadOfWhile implements Buffer {
        private final int[] items;
        private int head, count;

        public IfInsteadOfWhile(int capacity) { items = new int[capacity]; }

        @Override
        public synchronized void put(int e) throws InterruptedException {
            if (count == items.length) {                // BUG: after waking, the buffer may be full AGAIN
                wait();
            }
            if (count == items.length) {
                throw new IllegalStateException("put into a FULL buffer (woke up but condition no longer holds)");
            }
            items[(head + count) % items.length] = e;
            count++;
            notifyAll();
        }

        @Override
        public synchronized int take() throws InterruptedException {
            if (count == 0) {                           // BUG
                wait();
            }
            if (count == 0) {
                throw new IllegalStateException("take from an EMPTY buffer (woke up but condition no longer holds)");
            }
            int e = items[head];
            head = (head + 1) % items.length;
            count--;
            notifyAll();
            return e;
        }
    }

    /** Mistake 2: notify() instead of notifyAll() when producers AND consumers share one wait set. */
    public static final class NotifyInsteadOfNotifyAll implements Buffer {
        private final int[] items;
        private int head, count;

        public NotifyInsteadOfNotifyAll(int capacity) { items = new int[capacity]; }

        @Override
        public synchronized void put(int e) throws InterruptedException {
            while (count == items.length) {
                wait();
            }
            items[(head + count) % items.length] = e;
            count++;
            notify();                                   // BUG: may wake another PRODUCER, not a consumer
        }

        @Override
        public synchronized int take() throws InterruptedException {
            while (count == 0) {
                wait();
            }
            int e = items[head];
            head = (head + 1) % items.length;
            count--;
            notify();                                   // BUG: may wake another CONSUMER, not a producer
            return e;
        }
    }

    /** Adapter so the correct BoundedBuffer can run in the same harness. */
    public static final class Correct implements Buffer {
        private final BoundedBuffer<Integer> b;
        public Correct(int capacity) { b = new BoundedBuffer<>(capacity); }
        @Override public void put(int e) throws InterruptedException { b.put(e); }
        @Override public int take() throws InterruptedException { return b.take(); }
    }
}
```

### `BrokenBufferHarness.java`

```java
package com.mastery.conc.pc;

import com.mastery.conc.pc.BrokenBuffers.Buffer;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.IntFunction;

/**
 * Runs P producers and C consumers through a capacity-1 buffer many times and classifies each run:
 * OK, EXCEPTION (a broken invariant was detected), or HANG (no progress within the timeout:
 * every thread is waiting, nobody will ever notify).
 */
public final class BrokenBufferHarness {
    enum Outcome { OK, EXCEPTION, HANG }

    static Outcome runOnce(Buffer buffer, int producers, int consumers, int itemsEach) throws InterruptedException {
        int totalItems = producers * itemsEach;
        int perConsumer = totalItems / consumers;
        AtomicReference<Throwable> error = new AtomicReference<>();
        List<Thread> threads = new ArrayList<>();
        for (int p = 0; p < producers; p++) {
            threads.add(new Thread(() -> {
                try { for (int i = 0; i < itemsEach; i++) buffer.put(i); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                catch (RuntimeException e) { error.compareAndSet(null, e); }
            }));
        }
        for (int c = 0; c < consumers; c++) {
            threads.add(new Thread(() -> {
                try { for (int i = 0; i < perConsumer; i++) buffer.take(); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                catch (RuntimeException e) { error.compareAndSet(null, e); }
            }));
        }
        threads.forEach(Thread::start);
        long deadline = System.nanoTime() + 2_000_000_000L;           // 2 s is plenty for a correct run
        for (Thread t : threads) {
            long remainingMs = Math.max(1, (deadline - System.nanoTime()) / 1_000_000);
            t.join(remainingMs);
        }
        boolean stuck = threads.stream().anyMatch(Thread::isAlive);
        threads.forEach(Thread::interrupt);                          // release anything still waiting
        for (Thread t : threads) t.join();
        if (error.get() != null) return Outcome.EXCEPTION;
        return stuck ? Outcome.HANG : Outcome.OK;
    }

    public static void main(String[] args) throws InterruptedException {
        int runs = 50;
        record Variant(String name, IntFunction<Buffer> factory) { }
        List<Variant> variants = List.of(
                new Variant("correct (while + notifyAll)", BrokenBuffers.Correct::new),
                new Variant("if instead of while", BrokenBuffers.IfInsteadOfWhile::new),
                new Variant("notify instead of notifyAll", BrokenBuffers.NotifyInsteadOfNotifyAll::new));
        for (Variant v : variants) {
            int ok = 0, exception = 0, hang = 0;
            for (int r = 0; r < runs; r++) {
                switch (runOnce(v.factory().apply(1), 4, 4, 2_000)) {
                    case OK -> ok++;
                    case EXCEPTION -> exception++;
                    case HANG -> hang++;
                }
            }
            System.out.printf("%-30s %d runs: OK=%d  EXCEPTION=%d  HANG=%d%n", v.name(), runs, ok, exception, hang);
        }
    }
}
```

### Results (4 producers + 4 consumers × 2 000 items, capacity 1, 50 runs each)

```text
correct (while + notifyAll)    50 runs: OK=50  EXCEPTION=0  HANG=0
if instead of while            50 runs: OK=0  EXCEPTION=50  HANG=0
notify instead of notifyAll    50 runs: OK=0  EXCEPTION=0  HANG=50
```

Both mistakes failed in **every single run**, not "occasionally under load". (With bigger buffers and fewer threads, they fail much more rarely, which is exactly how such bugs survive testing and reach production.)

### One possible interleaving for `if` instead of `while` → EXCEPTION

```
buffer empty (count 0)
C1: take → count==0 → wait()          (releases the lock)
C2: take → count==0 → wait()
P1: put(x) → count=1 → notifyAll()    (wakes C1 and C2)
C1: re-acquires the lock first → `if` already passed → takes x → count=0
C2: re-acquires the lock → `if` already passed → proceeds with count==0 → ✗ "take from an EMPTY buffer"
```

With `while`, C2 would re-check `count == 0` and wait again.

### One possible interleaving for `notify` instead of `notifyAll` → HANG (lost wake-up)

(An illustrative trace consistent with the measured hangs; the harness shows *that* it hangs, and this shows *how* it can.)

```
capacity 1, buffer FULL; producers P1, P2 waiting ("not full")
C1: take → count=0 → notify() → wakes P1        ✓ correct kind
C1: take again → count==0 → wait()
C2: take → count==0 → wait()
P1: put → count=1 → notify() → wakes C1          ✓
P1: put again → full → wait()
C1: take → count=0 → notify() → wakes… C2 (another CONSUMER)  ✗ wrong kind
C1: take again → empty → wait()
C2: woke, re-checks: count==0 → wait()           the signal is swallowed; the producers are never woken
→ P1, P2, C1, C2 all in the wait set, nobody left to notify → HANG
```

---

## 4. Graceful Shutdown: Poison Pill vs Interruption

**Poison pill:** when producers finish, they put a special sentinel value; a consumer that takes it stops.

```java
static final Item POISON = new Item(-1, "POISON");    // compared by identity

// producer side, after all real items:
for (int i = 0; i < consumerCount; i++) buffer.put(POISON);   // ONE pill PER consumer

// consumer loop:
while (true) {
    Item it = buffer.take();
    if (it == POISON) break;
    process(it);
}
```

- ✅ Every item put *before* the pill is still processed (FIFO), so the shutdown is **clean**.
- ⚠️ You need **exactly one pill per consumer**. With several producers, coordinate who sends them (e.g. the last producer to finish). Too few pills leaves consumers waiting forever; too many leaves junk in the buffer.

**Interruption:** `consumerThread.interrupt()` makes the blocked `take()` throw `InterruptedException`.

- ✅ Works even when producers are stuck or gone.
- ⚠️ It's **abrupt**: items still in the buffer are abandoned unless the consumer drains them before exiting.

Typical choice: poison pills for "finish your work and stop", interruption for "cancel now" (and executors use interruption in `shutdownNow()`).

---

## 5. What `java.util.concurrent` Gives You

`ArrayBlockingQueue` is this exact design (a circular array + one lock), but it uses a `ReentrantLock` with **two `Condition`s** (`notEmpty`, `notFull`), so producers and consumers wait in separate queues and `signal()` (the analogue of `notify`) becomes safe and efficient. The rows 57–61 project builds that version from scratch, and this buffer is its starting point.

---

## Key Takeaways

1. **`wait()` always in a `while` loop** re-checking the condition predicate: competitors, spurious wake-ups, shared wait sets.
2. **`notifyAll()`** when different kinds of waiters share one wait set; `notify()` loses wake-ups (measured: **HANG in 50/50 runs**).
3. Use a **private lock**, so outside code can't steal or inject notifications.
4. Test blocking behaviour by **observing thread states**, not by sleeping; put **timeouts** on every concurrent test.
5. Prove exactly-once delivery with **unique ids**, a duplicate check and a count.
6. Shutdown: a **poison pill per consumer** (clean) or **interruption** (abrupt).
