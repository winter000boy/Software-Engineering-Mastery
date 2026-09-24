# #012 — Java Memory Model Overview & Stack vs Heap (Study)

**Date:** 2026-08-18  
**Day:** Catch-up Day 3 of 4

---

## At a Glance (copy to handwritten notes)

- **JMM (Java Memory Model)** = formal rules for how threads see reads/writes to shared memory — **not** the same as heap/stack layout.
- JMM solves: **visibility**, **ordering**, and **atomicity** across threads.
- Without JMM guarantees, CPU caches + compiler reordering break naive multi-threaded code.
- **`happens-before`** is the core JMM rule — if A happens-before B, B sees all writes A made.
- Key happens-before sources: **monitor lock/unlock**, **`volatile` read/write**, **thread start/join**, **transitivity**.
- **`volatile`** — visible to all threads immediately; no reordering around it; no compound atomicity (`count++` still broken).
- **`synchronized`** — mutual exclusion + flushes writes on unlock / loads on lock (full happens-before).
- **Stack vs heap in concurrency:** locals on stack are **thread-confined** (safe); shared mutable objects on heap need synchronization.
- **Publication problem:** making object visible to another thread without happens-before → other thread may see partial construction.
- JMM ≠ physical memory model — it's the **contract** between programmer and JVM/hardware.

---

## 1. Why the Java Memory Model Exists

### The problem

Modern CPUs and compilers optimize aggressively:

- **CPU caches** — each core has its own cache; writes may not be visible to other cores immediately.
- **Instruction reordering** — compiler and CPU reorder for performance if semantics preserved **within one thread**.
- **Write buffers** — stores delayed before hitting main memory.

Single-threaded code looks fine. Multi-threaded code without proper synchronization can fail **intermittently** — worst kind of bug.

### What JMM defines

The **Java Memory Model** (JLS Chapter 17, revised in Java 5+) specifies:

1. When a write by one thread is **visible** to a read by another.
2. Which **reorderings** are allowed.
3. Which operations are **atomic**.

> **Analogy:** JMM is the **traffic law** for threads sharing data. Heap/stack map is the **city layout** — related but different concerns.

---

## 2. JMM vs JVM Memory Areas

| Concept | Answers |
|---------|---------|
| **Heap / Stack / Metaspace** | Where bytes are stored physically in JVM process |
| **Java Memory Model** | Logical rules for thread interaction via shared variables |
| **Overlap** | Shared heap objects need JMM rules; stack locals are usually thread-confined |

```
Thread A stack          Thread B stack
(local vars — safe)     (local vars — safe)
      │                        │
      └──────► HEAP ◄──────────┘
            (shared objects —
             need JMM rules)
```

---

## 3. Core JMM Concepts

### 3.1 Main memory vs working memory (logical view)

Each thread has **working memory** (conceptual — maps to CPU registers/cache).  
Shared variables live in **main memory** (conceptual — maps to heap fields).

Thread must:
1. **Load** from main memory to working memory before use.
2. **Store** from working memory to main memory after change.

JVM may optimize actual loads/stores — JMM constrains what's allowed.

### 3.2 Visibility

**Visibility** = when thread B reads variable X, does it see thread A's latest write?

Without synchronization — **no guarantee**.

### 3.3 Ordering

**Ordering** = can operations appear to execute out of program order?

Allowed for performance **unless** happens-before prevents it.

### 3.4 Atomicity

**Atomicity** = operation completes as indivisible unit.

- Read/write to `int`, references, etc. (except `long`/`double` on 32-bit) — atomic.
- **`long`/`double` on 32-bit JVM** — may be torn (use `volatile`).
- **`i++`** — NOT atomic (read-modify-write = 3 steps).

---

## 4. Happens-Before — The Central Rule

### Definition

If action **A happens-before B**, then:
- B sees all writes made by A (and earlier actions).
- A is **ordered before** B from a memory perspective.

Happens-before is **partial order** — not all actions are ordered relative to each other.

### Established happens-before rules

| Rule | Meaning |
|------|---------|
| **Program order** | Within same thread, earlier statement hb later (as-if-serial) |
| **Monitor lock** | Unlock on monitor M **hb** subsequent lock on M |
| **Volatile** | Write to volatile V **hb** subsequent read of V |
| **Thread start** | `thread.start()` **hb** any action in started thread |
| **Thread join** | Any action in thread T **hb** `T.join()` returns |
| **Transitivity** | If A hb B and B hb C, then A hb C |
| **Final fields** | Safe publication of immutable objects (constructor finish hb read of final fields) |

### Diagram: lock happens-before

```
Thread A                    Thread B
────────                    ────────
write x = 1
synchronized(lock) {
  write y = 2
}  ── unlock ──────────►  synchronized(lock) {
                              read y → guaranteed 2
                              read x → guaranteed 1
                            }
```

---

## 5. Stack vs Heap — Concurrency Lens

### Stack — thread-confined (usually safe)

```java
void process() {
    int count = 0;           // stack — only this thread sees it
    List<String> local = new ArrayList<>();  // ref on stack; object on heap BUT
    local.add("a");          // not published to other threads → still safe
}
```

**Rule:** Local variables (primitives + references) live in stack frames. If reference **never escapes** the thread, even heap object is effectively confined.

### Heap — shared mutable state (needs care)

```java
class Counter {
    int count = 0;           // on heap — shared if same instance used by multiple threads
}

// Thread A and B both:
counter.count++;             // race — lost updates without sync
```

| Storage | Thread safety default |
|---------|----------------------|
| Local primitive | Safe (thread-confined) |
| Local reference, no escape | Safe |
| Instance/static field on shared object | **Unsafe** if mutated without sync |
| Immutable object published safely | Safe to read |

### Escape analysis (brief)

JVM may prove object never escapes thread → allocate on **stack** (scalar replacement) — optimization, not something you rely on in code design.

---

## 6. Visibility Failures — Classic Examples

### 6.1 Stale flag (no volatile)

```java
class Worker implements Runnable {
    private boolean running = true;  // heap field

    public void run() {
        while (running) { doWork(); }  // may loop forever!
    }

    public void stop() { running = false; }  // other thread may never see false
}
```

**Fix:** `private volatile boolean running = true;`

### 6.2 Double-checked locking (broken without volatile)

```java
class Singleton {
    private static Singleton instance;  // broken in old Java without volatile

    static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null)
                    instance = new Singleton();  // reordering can expose half-built object
            }
        }
        return instance;
    }
}
```

**Fix:** `private static volatile Singleton instance;`  
Or use **enum singleton** / **holder idiom** (Effective Java Item 3).

### 6.3 Partially constructed object

```java
// Thread A
this.field = 42;        // step 1
// constructor not finished — Thread B reads object
// may see field = 0 (default) due to visibility/reordering
```

**Fix:** Safe publication via `volatile` ref, synchronized, or final fields.

---

## 7. Synchronization Mechanisms & JMM

### volatile

| Guarantees | Does NOT guarantee |
|------------|-------------------|
| Visibility on read/write | Atomicity of compound ops |
| Prevents reordering with adjacent volatiles | Mutual exclusion |
| Cheaper than lock for single read/write | |

```java
private volatile int status;

void setReady() { status = READY; }  // visible immediately
int getStatus() { return status; }    // reads latest
```

### synchronized

```java
synchronized (lock) {
    count++;   // mutual exclusion + visibility
}
```

- **Unlock happens-before** next lock — all writes visible.
- Also provides **mutual exclusion** — only one thread in block.

### java.util.concurrent (JMM-aware)

| Tool | JMM role |
|------|----------|
| `AtomicInteger` | CAS + volatile-like visibility |
| `ConcurrentHashMap` | Thread-safe without locking entire map |
| `CountDownLatch`, `Semaphore` | Happens-before on await/release |
| `BlockingQueue` | Safe handoff between threads |

---

## 8. Safe Publication Idioms

Publishing = making object reference visible to another thread.

| Idiom | How it works |
|-------|--------------|
| Static initializer | JVM initializes class once, thread-safe |
| `volatile` reference field | Write to volatile hb read |
| `synchronized` access | Lock hb |
| `final` fields | Constructor completion hb read of finals (immutable objects) |
| Concurrent collections | Internal happens-before |

```java
// Static holder — lazy, thread-safe (Effective Java)
class Holder {
    static class LazyHolder {
        static final Expensive INSTANCE = new Expensive();
    }
    static Expensive getInstance() { return LazyHolder.INSTANCE; }
}
```

---

## 9. Stack vs Heap — Complete Comparison Table

| Dimension | Stack | Heap |
|-----------|-------|------|
| **Scope** | Per thread | All threads |
| **Stores** | Frames, locals, refs | Objects, arrays |
| **Lifetime** | Method call duration | Until GC collects (no refs) |
| **Allocation speed** | Very fast | Slower (GC bookkeeping) |
| **Size limit** | Small (`-Xss`) | Large (`-Xmx`) |
| **GC** | Automatic on pop | Garbage collector |
| **Concurrency** | Thread-confined → safe | Shared → needs JMM |
| **Errors** | StackOverflowError | OutOfMemoryError |
| **Typical misuse** | Deep recursion | Memory leaks, races on shared mutable state |

### Mental model for handwritten notes

```
STACK = "where am I in the call chain + my local scratch space"
HEAP  = "the shared warehouse of objects"

JMM   = "rules for when warehouse changes are visible to other workers"
```

---

## 10. JMM and the `long` / `double` Special Case

On **32-bit** JVMs, 64-bit `long`/`double` reads/writes may be **torn** (two 32-bit halves).  
On **64-bit** JVMs, JVM spec still allows torn reads in theory for non-volatile fields — use `volatile` for cross-thread 64-bit flags/counters if correctness critical.

---

## 11. What JMM Does NOT Specify

- Physical layout of heap generations
- Stack size defaults
- GC algorithm choice
- Class loading details

Those are **JVM implementation** details, not JMM.

---

## 12. Pitfalls & Interview Angles

### Common mistakes

1. **Assuming `int` increment is thread-safe** — `count++` is read-modify-write.
2. **Using volatile for everything** — doesn't compose; use atomics or locks for compound ops.
3. **Confusing JMM with "memory leak"** — leaks are heap lifecycle; JMM is visibility/ordering.
4. **Thinking stack variables are always safe** — if reference **escapes** (stored in static field), heap object becomes shared.

### Interview questions

| Q | A |
|---|---|
| What is JMM? | Spec for visibility, ordering, atomicity across threads |
| happens-before? | If A hb B, B sees A's writes; ordering guarantee |
| volatile vs synchronized? | volatile: visibility only; synchronized: visibility + mutual exclusion |
| Why stale values? | CPU cache, no flush without hb edge |
| Stack vs heap thread safety? | Stack locals confined; heap shared mutable needs sync |
| Safe publication? | volatiles, locks, finals, static init, concurrent utilities |
| Is JMM same as heap? | No — logical concurrency model vs physical storage |

---

## Handwritten Cheat Sheet

```
JMM = visibility + ordering + atomicity (NOT heap/stack layout)

Problem: CPU cache, compiler reorder → threads see stale/wrong data
Solution: happens-before edges

happens-before sources:
  program order (same thread)
  unlock → lock (same monitor)
  volatile write → volatile read
  thread.start() → thread actions
  thread actions → thread.join() returns
  transitivity

volatile: visible reads/writes, no reorder; NOT atomic for i++
synchronized: lock + unlock = hb + mutual exclusion

Stack: thread-confined locals → safe by default
Heap: shared objects → volatile/sync/atomic if mutated

Safe publish: static init, volatile ref, synchronized, final fields
Broken: DCL without volatile, non-volatile stop flag

count++ = read + add + write → race without sync
Use AtomicInteger or synchronized for compound ops
```

---

**Next note:** #013 — Java Memory Model & stack vs heap (practice/apply)
