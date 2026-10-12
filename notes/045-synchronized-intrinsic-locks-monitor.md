# #045 — `synchronized`, Intrinsic Locks, Monitors

**Date:** 2026-10-12  
**Roadmap row:** 45 — P1 Java Core · Concurrency & Multithreading I · *Learn*  
**Sources:** JLS §17.1 (synchronization), JVM spec (`monitorenter`/`monitorexit`, `ACC_SYNCHRONIZED`), *Java Concurrency in Practice* §2.3–2.4, §3.1, JEP 444 / JEP 491

> Every ✔︎ result was produced on JDK 21.0.10 (Apple M4), including the `javap` bytecode and the `jstack` thread dump shown below.

---

## At a Glance (copy to handwritten notes)

- **Every Java object has an intrinsic lock** (its **monitor**). `synchronized` acquires it on entry and releases it on exit, **including on exceptions** ✔︎.
- `synchronized` gives two guarantees:
  1. **Mutual exclusion:** at most one thread at a time holds a given monitor.
  2. **Visibility:** *unlock happens-before every later lock of the same monitor*. Everything written before the unlock is visible after the lock (#012).
- **Which lock?** Instance method → `this` · static method → `TheClass.class` · block → the object you name. **The instance lock and the class lock are different locks** ✔︎.
- **Reentrant:** a thread that already holds a monitor can re-enter it (so a `synchronized` method can call another `synchronized` method on the same object) ✔︎.
- Bytecode ✔︎: a block compiles to **`monitorenter` + two `monitorexit`s** (normal exit and the exception path); a method gets the **`ACC_SYNCHRONIZED`** flag.
- A thread waiting to enter is **`BLOCKED`**, and `jstack` shows *"waiting to lock <0x…>"* next to the holder's *"locked <0x…>"* ✔︎.
- **Lock on a `private final` object.** Never on `Integer`, `String` literals or other shared or interned objects: `Integer.valueOf(42) == Integer.valueOf(42)`, so "unrelated" code **blocks** ✔︎.
- A **monitor = lock + wait set**; `wait`/`notify` use the wait set (row 46).
- **JDK 21 virtual threads:** blocking inside `synchronized` **pins** the carrier. ✔︎ 200 virtual threads sleeping 100 ms under their *own* lock: **2 081 ms** with `synchronized` vs **104 ms** with `ReentrantLock`. Fixed in JDK 24 (JEP 491).

---

## 1. Why We Need Locks

```java
class Counter {
    private int count;
    void increment() { count++; }      // read → add → write: three steps, NOT atomic
}
```

Two threads can both read 5 and both write 6, so an increment is **lost**. That's a **race condition** (read-modify-write); row 49 reproduces it with real numbers. `synchronized` makes the three steps behave as one **atomic** unit with respect to the other threads that synchronize on the **same** lock.

> JCIP: *"For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is **guarded by** that lock."*

The key words are **all accesses** (reads too) and **the same lock**.

---

## 2. The Three Forms

```java
public class Account {
    private final Object lock = new Object();     // a private lock object (preferred)
    private long balance;
    private static int instances;

    public synchronized void deposit(long amt) {  // locks `this`
        balance += amt;
    }

    public void withdraw(long amt) {
        synchronized (lock) {                     // locks `lock`: the smallest critical section
            if (amt > balance) throw new IllegalStateException("insufficient funds");
            balance -= amt;
        }
    }

    public static synchronized void created() {   // locks Account.class
        instances++;
    }
}
```

| Form | Lock acquired | Note |
|------|---------------|------|
| `synchronized` instance method | `this` | The lock is **public**: any code holding a reference can `synchronized (account)` and interfere |
| `static synchronized` method | `Account.class` | One lock for the whole class, shared with any code that locks `Account.class` |
| `synchronized (expr) { … }` block | The object `expr` evaluates to | Lets you pick a **private** lock and keep the critical section **small** |

✔︎ **Instance vs class lock:** while one thread held `Monitors.class` (inside a `static synchronized` region), another thread entered a `synchronized` **instance** method on the same class **immediately** (2 µs). Different monitors, so no exclusion. A static field guarded by an instance lock is therefore **not** protected.

> ⚠️ Mixing forms is a classic bug: `deposit` locks `this`, `withdraw` locks `lock`, so they **don't** exclude each other and `balance` is a data race. **Pick one lock per piece of state** and use it for every access (in the example above both must use `lock`).

---

## 3. What `synchronized` Compiles To ✔︎

### Block → `monitorenter` / `monitorexit`

```java
void syncBlock() { synchronized (lock) { count++; } }
```

```text
$ javap -c -p Monitors.class
  void syncBlock();
       0: aload_0
       1: getfield      #7     // Field lock:Ljava/lang/Object;
       4: dup
       5: astore_1                ← keep the lock reference in a local (in case the field changes)
       6: monitorenter            ← acquire
       7: aload_0
       8: dup
       9: getfield      #13    // Field count:I
      12: iconst_1
      13: iadd
      14: putfield      #13    // Field count:I
      17: aload_1
      18: monitorexit             ← release on the normal path
      19: goto          27
      22: astore_2
      23: aload_1
      24: monitorexit             ← release on the EXCEPTION path
      25: aload_2
      26: athrow                  ← rethrow
      27: return
    Exception table:
       from    to  target type
           7    19    22   any    ← any exception in the body → jump to 22
          22    25    22   any
```

The compiler generates a hidden `finally`. That's why ✔︎ another thread could acquire the lock right after an exception escaped a synchronized block: **the lock is always released**. (`ReentrantLock` doesn't do this for you, which is why you always write `lock(); try { … } finally { unlock(); }`, row 57+.)

### Method → `ACC_SYNCHRONIZED` flag

```text
$ javap -v -p Monitors.class
  synchronized void syncMethod();
    flags: (0x0020) ACC_SYNCHRONIZED
  static synchronized void staticSync();
    flags: (0x0028) ACC_STATIC, ACC_SYNCHRONIZED
```

No `monitorenter` in the body: the **JVM** acquires the monitor (on `this`, or on the `Class` object for static methods) when invoking the method, and releases it on any return or throw.

---

## 4. Reentrancy ✔︎

```java
synchronized void outer() { inner(); }    // holds `this`
synchronized void inner() { … }           // needs `this` again → allowed
// ✔︎ "reentrant: inner() entered while holding the same lock; holdsLock=true"
```

The JVM tracks an **owner thread + an acquisition count** per monitor: re-entry increments it, each exit decrements it, and the lock is free at 0. Without reentrancy, a subclass's `synchronized` override calling `super.method()` (also synchronized) would **deadlock with itself** (JCIP §2.3.2).

`Thread.holdsLock(obj)` tells you whether the **current** thread holds `obj`'s monitor. It's useful in `assert` statements.

---

## 5. Visibility: The Other Half of `synchronized`

Locking isn't only about mutual exclusion. Without it, one thread **may never see** another's writes (stale values, compiler hoisting, CPU caches; #012 §6, row 48).

```
Thread A                                    Thread B
x = 1;                                      synchronized (m) {     ← lock m
synchronized (m) {  ← lock m                    read x  // guaranteed to see 1
    y = 2;                                      read y  // guaranteed to see 2
}                   ← unlock m ──happens-before──▶ (B locked m AFTER A unlocked it)
```

> JCIP §3.1.3: *"Locking is not just about mutual exclusion; it is also about memory visibility. To ensure that all threads see the most up-to-date values of shared mutable variables, the reading and writing threads must synchronize on a common lock."*

So a **getter** for a field guarded by a lock must also take the lock. `synchronized` only on the setter isn't enough, since the reader may see a stale value (JCIP's `SynchronizedInteger`). The exception is a `volatile` field, when the write doesn't depend on the current value (row 48).

---

## 6. Monitor = Lock + Wait Set

```
            ┌───────────────────── monitor of object m ─────────────────────┐
  entry set │  threads BLOCKED trying to enter synchronized(m)              │
  ─────────▶│                                                                │
            │      OWNER: at most one thread inside synchronized(m)          │
            │                                                                │
  wait set  │  threads that called m.wait(): WAITING, monitor RELEASED       │
            │  m.notify() moves one of them back to compete for the lock;    │
            │  m.notifyAll() moves all of them                                │
            └────────────────────────────────────────────────────────────────┘
```

- `wait()`, `notify()` and `notifyAll()` **must be called while holding that object's monitor**, or they throw `IllegalMonitorStateException`.
- `wait()` **releases** the monitor and suspends; when woken, it must **re-acquire** the monitor before returning. That's why you always re-check the condition in a `while` loop (row 46).
- The **Java monitor pattern** (JCIP §4.2.1): encapsulate all mutable state in an object and guard it with that object's own intrinsic lock. `Vector`, `Hashtable` and `Collections.synchronizedList` work this way.

---

## 7. Choosing the Lock Object

| Lock on… | Verdict | Why |
|----------|:---:|-----|
| `private final Object lock = new Object()` | ✅ | Nobody else can get the reference, so no outside interference and no accidental sharing |
| `this` (synchronized methods) | ⚠️ OK for simple classes | Public: clients *can* lock it (sometimes intended, e.g. client-side locking; JCIP §4.4.1) |
| `SomeClass.class` | ⚠️ | Global to the class; any code anywhere can lock it |
| **`Integer`, `Long`, `Boolean`** | ❌ | ✔︎ `Integer.valueOf(42) == Integer.valueOf(42)` (the cache −128..127): "unrelated" components share one lock, and the second thread was **BLOCKED** |
| **`String` literals** | ❌ | ✔︎ `"config" == "config"` (interned): the same trap across the whole JVM, including libraries |
| A **non-final field** | ❌ | If the field is reassigned, threads lock *different* objects, so no mutual exclusion |
| A value-based class (`Optional`, `LocalDate`, records you didn't design for it) | ❌ | Identity isn't guaranteed; `javac` warns (`-Xlint:synchronization`) |

---

## 8. Seeing Locks in a Thread Dump ✔︎

A real dump (`jstack <pid>`) while one thread holds a lock and sleeps, and another tries to enter:

```text
"order-writer" #20 ... waiting on condition
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@21.0.10/Thread.java:509)
	at Dump.lambda$main$0(Dump.java:4)
	- locked <0x0000000697c14618> (a java.lang.Object)            ← the HOLDER

"order-reader" #21 ... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Dump.lambda$main$1(Dump.java:6)
	- waiting to lock <0x0000000697c14618> (a java.lang.Object)   ← the same address
```

**How to read a hang:** find the `BLOCKED` threads → note the `<0x…>` address they're "waiting to lock" → search for "locked <0x…>" to find the holder → see what the holder is doing (here: **sleeping while holding a lock**, a bug in itself). If two threads each hold the lock the other wants, `jstack` prints **"Found one Java-level deadlock"** (rows 54–55).

**Naming threads** (`new Thread(task, "order-writer")`) makes dumps readable. Always name them, or give pools a thread factory that does.

---

## 9. Cost & Implementation Notes

- An **uncontended** lock is cheap: a CAS on the object's header (the "mark word", #004). Under **contention**, the JVM inflates it to a full monitor and parks the waiting threads, which means OS involvement and context switches, so it costs far more.
- **Biased locking** (an old optimisation for single-thread locking) was disabled by default and deprecated in JDK 15 (JEP 374), then removed in JDK 18. Don't tune for it.
- **Keep critical sections short.** Never do I/O, sleeps or calls into unknown code (callbacks, listeners) while holding a lock. That causes contention at best and deadlock at worst (the #035 listener runs under the decorator's lock, which is documented as a limitation).
- **Lock coarsening / elision:** the JIT may merge adjacent synchronized blocks, or drop locks on objects that never escape a thread (escape analysis). That's another reason micro-benchmarks of locks need JMH.

---

## 10. Virtual Threads and `synchronized` (JDK 21) ✔︎

A virtual thread that **blocks while holding a monitor** can't unmount from its carrier: it's **pinned**, and the carrier stays busy doing nothing.

```text
200 virtual threads, each sleeps 100 ms while holding its OWN lock (zero contention):
synchronized     2 081 ms   ← pinned: only 10 carriers (= cores) → 200/10 × 100 ms ≈ 2 s
ReentrantLock      104 ms   ← unmounts while sleeping → all 200 sleep concurrently

$ java -Djdk.tracePinnedThreads=short Pinning.java
VirtualThread[#34]/runnable@ForkJoinPool-1-worker-6 reason:MONITOR
    Pinning.lambda$main$0(Pinning.java:16) <== monitors:1
```

**Rule for JDK 21–23:** with virtual threads, don't block (I/O, sleep, waiting) inside `synchronized`; use `ReentrantLock` there. Short, non-blocking `synchronized` sections are fine. **JDK 24 (JEP 491)** lets virtual threads unmount while holding monitors, which removes most pinning.

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| What does `synchronized` guarantee? | Mutual exclusion **and** visibility (unlock happens-before the next lock of the same monitor) |
| Which lock does a static synchronized method use? | The `Class` object, which is different from instance locks |
| Is `synchronized` reentrant? | Yes: an owner + count per monitor |
| What if an exception is thrown inside? | The lock is released (compiler-generated `monitorexit` on the exception path / the JVM for methods) |
| Why synchronize the getter too? | Visibility: an unsynchronized read may see a stale value |
| Why not lock on a String or an Integer? | Interning and caching make them shared across unrelated code, causing contention or deadlock |
| BLOCKED vs WAITING? | BLOCKED: trying to enter a monitor. WAITING: inside `wait()` (monitor released) |
| What's a monitor? | A lock + a wait set (+ entry set); `wait`/`notify` need the monitor held |
| `synchronized` method vs block? | A block can use a private lock and keep the critical section small |
| `synchronized` and virtual threads? | JDK 21: blocking inside it pins the carrier; use `ReentrantLock`, or JDK 24+ |

---

## Handwritten Cheat Sheet

```
synchronized = MUTUAL EXCLUSION + VISIBILITY (unlock hb next lock, same monitor)
LOCKS: instance method → this · static → Class · block → any object (prefer private final)
instance lock ≠ class lock (static fields need the class lock or one shared lock)
REENTRANT: owner + count · Thread.holdsLock(obj)
BYTECODE: block = monitorenter + 2× monitorexit (normal + exception path) · method = ACC_SYNCHRONIZED
exception inside → lock released automatically
GUARDED BY: every access (reads too!) under the SAME lock
NEVER lock on: Integer/Long/Boolean (cache) · String literals (interned) · non-final fields
MONITOR = lock + wait set; wait/notify need the monitor held, else IllegalMonitorStateException
jstack: BLOCKED "waiting to lock <0x…>" ↔ "- locked <0x…>" → find the holder
short critical sections; no I/O / sleep / callbacks while holding a lock
VIRTUAL THREADS (21): blocking in synchronized PINS → 2 081 ms vs 104 ms (ReentrantLock); JDK 24 fixes
```

---

**Next:** row 46 — Producer-consumer with wait/notify (#046)
