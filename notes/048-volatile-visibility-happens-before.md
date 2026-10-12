# #048 — `volatile`, Visibility, Happens-Before

**Date:** 2026-10-12  
**Roadmap row:** 48 — P1 Java Core · Concurrency & Multithreading I · *Learn*  
**Sources:** JLS §17.4 (memory model), §17.7 (64-bit writes); *Java Concurrency in Practice* §3.1–3.5, §16.1–16.2  
**Builds on:** #012 (JMM overview), #013 (practice), #045 (synchronized). **Deep dive later:** rows 62–63

> Everything marked ✔︎ was reproduced on JDK 21.0.10, Apple M4 (ARM, a weakly ordered CPU), in several runs.

---

## At a Glance (copy to handwritten notes)

- Without synchronization, a thread may **never** see another thread's write. ✔︎ A spin loop on a plain `boolean` **never stopped** (still `RUNNABLE` 2 s after the write); with `volatile` it stopped in **0.08–0.1 ms**.
- The cause in that demo was the **JIT** hoisting the read out of the loop. ✔︎ Under `-Xint` (interpreter only), the same plain loop stops. The JMM *allows* this optimisation.
- **Reordering is real, too.** ✔︎ Store-buffering litmus test, `x=1; r1=y ‖ y=1; r2=x`: "both read 0" (impossible in any interleaving) happened in **1.1–1.5 million of 5 million** rounds with plain fields, and **0** with volatile semantics.
- **`volatile` gives:** visibility (a volatile write happens-before every later read of that variable, plus everything written before it) **and** ordering (no reordering across it). It does **not** give **atomicity**: ✔︎ `volatile count++` lost ~2.4M of 4M increments.
- **Use `volatile` when** writes don't depend on the current value (or there's a single writer) *and* the variable isn't part of an invariant with other variables. Typical uses: **stop flags, status fields, publishing an immutable object**.
- For read-modify-write, use **`Atomic*`** (CAS) or **locks**.
- **Double-checked locking** needs the field to be `volatile`. Better still, use the **holder-class idiom** or an enum.
- **Happens-before sources:** program order, monitor unlock→lock, volatile write→read, `Thread.start`, `join`, final-field freeze, `java.util.concurrent` hand-offs, and **transitivity**.

---

## 1. Visibility: The Problem ✔︎

```java
static boolean stop;                          // plain field

Thread spinner = new Thread(() -> { while (!stop) { } });
spinner.start();
Thread.sleep(500);                            // let the JIT compile the loop
stop = true;                                  // main thread writes
spinner.join(2_000);
// ✔︎ spinner.isAlive() == true, state RUNNABLE: it NEVER saw stop == true
```

| Run | Plain `boolean` | `volatile boolean` |
|-----|-----------------|--------------------|
| JIT (default) | **Never stops** ✔︎ (twice out of two) | Stops in **0.095 ms / 0.078 ms** ✔︎ |
| `-Xint` (interpreter) | Stops ✔︎ | — |

**Why:** with no happens-before edge between the write and the read, the JMM lets the compiler assume no other thread changes `stop`. So C2 turns `while (!stop) {}` into `if (!stop) while (true) {}`: it reads once and loops forever. Hardware caches aren't even needed to produce the bug. This is JCIP's `NoVisibility` example made real.

**Never** rely on "it'll be seen eventually": with plain fields, there is no "eventually".

---

## 2. Reordering: The Store-Buffering Litmus Test ✔︎

```
initially x = y = 0
Thread 1          Thread 2
x = 1;            y = 1;
r1 = y;           r2 = x;
```

In **any** interleaving of these four statements, at least one thread reads a 1. So `r1 == 0 && r2 == 0` is impossible under *sequential consistency*.

Measured (5 000 000 rounds, threads released together by spinning on volatile counters):

| Accesses | `r1 == 0 && r2 == 0` |
|----------|----------------------|
| Plain `int` fields | **1 114 697** and **1 515 233** (two runs, 22–30%) ✔︎ |
| Volatile semantics (`AtomicIntegerArray.set/get`) | **0** ✔︎ |

**Why:** each core's write first goes into its **store buffer**; the subsequent read of the *other* variable can complete before the buffered write becomes visible to the other core. ARM (and x86, in this specific pattern) allows this; the JIT may also reorder independent plain accesses. Volatile accesses are **sequentially consistent** in Java, so the JVM inserts the barriers that forbid it.

**Measurement lesson:** a first attempt that used a `CyclicBarrier` per round found **0 in 2 000 000**. The barrier itself is heavy synchronisation, and the racy windows rarely overlapped. *Not observing a race doesn't prove its absence.* The professional tool for this is **jcstress** (OpenJDK's concurrency stress-testing harness), covered in rows 62–63.

---

## 3. What `volatile` Guarantees

```java
volatile boolean ready;
int data;                       // plain

// Thread A                      // Thread B
data = 42;                       if (ready) {
ready = true;  // volatile WRITE     print(data);   // ✔ guaranteed 42
                                 }  // volatile READ of `ready` saw true
```

1. **Visibility:** a write to a volatile variable *happens-before* every subsequent read of that variable.
2. **"Piggybacking" ordering:** everything thread A did **before** the volatile write (`data = 42`) is visible to thread B **after** it reads the volatile and sees the new value. The volatile acts as a **publication fence**.
3. **No reordering across it:** writes before a volatile write can't move after it; reads after a volatile read can't move before it.
4. **Atomic reads and writes of `long`/`double`:** for non-volatile 64-bit fields, the JLS (§17.7) allows a write to be split into two 32-bit halves. Real 64-bit JVMs don't do that, but only `volatile` guarantees it.

### What it does NOT guarantee: atomicity ✔︎

```text
4 threads × 1 000 000 × count++
plain int        →   1 228 590 /   987 634   (lost ~2.8–3.0 M)
volatile int     →   1 552 634 / 1 569 356   (lost ~2.4 M)    ← volatile doesn't help
AtomicInteger    →   4 000 000 / 4 000 000   (lost 0)
```

`count++` is **read → add → write**. Volatile makes each read and each write visible, but two threads can still both read 7 and both write 8. Atomicity needs **CAS** (`AtomicInteger.incrementAndGet`) or a **lock** (row 49).

---

## 4. When `volatile` Is Enough (JCIP §3.1.4)

Use `volatile` only if **all** of these hold:

1. Writes don't depend on the current value, **or** only one thread ever writes.
2. The variable doesn't take part in an **invariant** with other state variables.
3. Locking isn't needed for any other reason while the variable is accessed.

| Use case | `volatile` OK? |
|----------|:---:|
| Shutdown/stop flag (`volatile boolean running`) | ✅ (the canonical use) |
| Status written by one thread, read by many (`volatile State state`) | ✅ |
| Publishing an **immutable** object (`volatile Config config = newConfig`) | ✅ |
| A counter incremented by several threads | ❌ use `AtomicLong` / `LongAdder` |
| Check-then-act (`if (x == null) x = create()`) | ❌ use a lock or `compareAndSet` |
| Two fields that must change together (`lower ≤ upper`) | ❌ use a lock, or one immutable object in one volatile field |

**Trick for multi-field invariants:** put the fields in an **immutable** object and publish it through **one** volatile (or `AtomicReference`). Readers then always see a consistent pair (JCIP's `VolatileCachedFactorizer`).

---

## 5. `volatile` vs `synchronized` vs `Atomic*`

| | `volatile` | `synchronized` / `Lock` | `AtomicInteger` etc. |
|---|:---:|:---:|:---:|
| Visibility | ✅ | ✅ | ✅ (volatile semantics) |
| Ordering | ✅ | ✅ | ✅ |
| Atomic read-modify-write | ❌ | ✅ (any block of code) | ✅ (single variable: `incrementAndGet`, `compareAndSet`, `updateAndGet`) |
| Multi-variable invariants | ❌ | ✅ | ❌ (one variable each) |
| Can block / deadlock | Never | Yes | Never (CAS retries instead) |
| Cost when uncontended | A barrier | CAS + barrier | CAS |

---

## 6. Double-Checked Locking (DCL)

```java
// ❌ BROKEN without volatile
class Broken {
    private static Helper helper;
    static Helper get() {
        if (helper == null) {                 // 1st check: no lock (fast path)
            synchronized (Broken.class) {
                if (helper == null) {         // 2nd check: under the lock
                    helper = new Helper();    // allocate → construct → publish may be REORDERED
                }
            }
        }
        return helper;                        // may return a reference to a not-yet-constructed object
    }
}
```

The reader on the fast path never takes the lock, so there's **no happens-before** with the writer. It can see `helper != null` *before* the constructor's writes are visible, and get a half-initialised object.

```java
// ✅ Fixed: volatile publication (Java 5+ memory model)
private static volatile Helper helper;
static Helper get() {
    Helper h = helper;                        // one volatile read on the fast path
    if (h == null) {
        synchronized (Fixed.class) {
            h = helper;
            if (h == null) {
                helper = h = new Helper();
            }
        }
    }
    return h;
}

// ✅✅ Simpler: the lazy initialisation holder class idiom (EJ Item 83)
class Lazy {
    private static class Holder { static final Helper INSTANCE = new Helper(); }
    static Helper get() { return Holder.INSTANCE; }   // class init is lazy AND thread-safe (#004 class loading)
}
```

The holder idiom relies on the JVM's guarantee that class initialisation runs **once**, under a lock, and is safely published to every thread that uses the class. It needs no volatile and has no fast-path cost.

---

## 7. The Happens-Before Rules (consolidated)

If action A *happens-before* action B, then B sees A's effects (and everything before A). The sources:

| Rule | Edge |
|------|------|
| **Program order** | Each action in a thread hb every later action in **that** thread |
| **Monitor lock** | An unlock of monitor M hb every **later** lock of M (#045) |
| **Volatile** | A write to volatile v hb every **later** read of v |
| **Thread start** | `t.start()` hb every action in t |
| **Thread termination** | Every action in t hb another thread detecting t's end (`join()` returns, `isAlive()` false) |
| **Interruption** | `t.interrupt()` hb t detecting the interrupt |
| **Finalizer** | The end of a constructor hb the start of the finalizer |
| **Transitivity** | A hb B and B hb C ⇒ A hb C |
| **Final fields** | Writes to `final` fields in a constructor are visible to any thread that obtains the reference *after* the constructor finishes, provided `this` didn't escape during construction |
| **`java.util.concurrent`** | Putting into a `BlockingQueue`/`ConcurrentMap` hb taking or reading it; a `CountDownLatch.countDown` hb a returning `await`; `Future` completion hb `get()`; and so on (per their Javadoc) |

**Practical reading:** if two threads access the same variable and at least one access is a write, and you can't name the chain of hb edges between them, **you have a data race**. Every correct program in this repo can name its edge: `join` (#043), the monitor (#046), a volatile flag (§3), `AtomicLong` (#043 DYNAMIC).

---

## 8. Safe Publication Idioms (JCIP §3.5.3)

To share an object safely, publish its reference with a happens-before edge:

1. Initialise it in a **static initialiser** (holder idiom).
2. Store it in a **`volatile`** field or an **`AtomicReference`**.
3. Store it in a **`final`** field of a properly constructed object.
4. Store it in a field **guarded by a lock**.
5. Put it into a **thread-safe collection** (`ConcurrentHashMap`, `BlockingQueue`, `CopyOnWriteArrayList`…).

**Immutable objects** (all fields `final`, no `this` escape) are safe even if published through a data race (the final-field guarantee). That's one more reason for EJ Item 17's "immutable by default" (#003).

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| What does `volatile` guarantee? | Visibility + ordering (sequential consistency for volatile accesses); not atomicity |
| Why can a loop on a non-volatile flag spin forever? | No happens-before ⇒ the JIT may hoist the read out of the loop (✔︎ reproduced; stops under `-Xint`) |
| Is `volatile int count; count++` thread-safe? | No: read-modify-write (✔︎ lost ~60% of increments) |
| When is `volatile` sufficient? | Single writer or value-independent writes, not part of an invariant |
| Why does DCL need `volatile`? | Without it, readers may see the reference before the constructor's writes |
| Better than DCL? | The holder-class idiom, or an enum singleton |
| What is happens-before? | A JMM ordering guarantee: if A hb B, B sees A's effects |
| Can "r1 == 0 && r2 == 0" happen in the store-buffering test? | Yes with plain fields (✔︎ 22–30% of rounds on M4); never with volatile |
| How do you test for such races properly? | jcstress, not ad-hoc loops (an ad-hoc barrier version saw nothing) |

---

## Handwritten Cheat Sheet

```
NO hb ⇒ maybe NEVER visible: plain flag spin loop hung (JIT hoist), -Xint stopped, volatile 0.1 ms
REORDERING: x=1;r1=y || y=1;r2=x → both 0 in 22–30% (plain, M4) · 0 with volatile
VOLATILE = visibility + ordering (+ atomic long/double); NOT atomic: volatile count++ lost ~2.4M/4M
  write hb later read; everything before the write is visible after the read (publication)
USE: stop flags · status · publish an IMMUTABLE object (multi-field state → one immutable object)
DON'T: counters (Atomic*/LongAdder) · check-then-act (lock/CAS) · invariants across fields (lock)
DCL needs volatile; prefer holder idiom: class Holder { static final X I = new X(); }
HB RULES: program order · unlock→lock · volatile w→r · start · join · interrupt · final fields ·
          j.u.c hand-offs · transitivity
SAFE PUBLICATION: static init · volatile/AtomicReference · final field · lock · concurrent collection
TESTING races: absence of evidence ≠ evidence → jcstress
```

---

**Next:** row 49 — Reproduce then fix a race condition (#049)
