# #013 — Java Memory Model & Stack vs Heap (Practice / Apply)

**Date:** 2026-08-18  
**Day:** Catch-up Day 4 of 4

---

## At a Glance (copy to handwritten notes)

- **Practice goal:** trace where data lives (stack/heap) and whether code is JMM-correct.
- **Thread confinement** — safest pattern; no shared mutable heap state.
- **Fix visibility** with `volatile`; **fix atomicity** with `synchronized` or `Atomic*`.
- **`count++` on shared field** — always wrong without synchronization.
- **StackOverflowError** — reproduce with recursion; **heap OOM** — grow collections.
- **Escape analysis exercise** — if reference stored in static field, object is shared.
- **happens-before drill:** lock/unlock, volatile write/read, start/join.
- Run with `-verbose:gc`, `-XX:+PrintCompilation` to connect theory to runtime (optional).
- Self-check: for each variable ask — **where?** (stack/heap) and **who else can see it?**

---

## Part A — Stack vs Heap Tracing Exercises

### Exercise 1: Trace memory locations

**Code:**

```java
public class MemoryTrace {
    static int staticCounter = 0;          // (1)
    int instanceField = 42;              // (2)

    public static void main(String[] args) {
        int localPrimitive = 10;         // (3)
        MemoryTrace obj = new MemoryTrace(); // (4)
        obj.instanceField = 99;
        staticCounter++;
        process(localPrimitive, obj);
    }

    static void process(int param, MemoryTrace ref) {  // (5)
        int localInMethod = param + 1;   // (6)
        ref.instanceField = localInMethod;
    }
}
```

**Your task (fill before peeking at answers):**

| Variable / object | Stack or Heap? | Notes |
|-------------------|----------------|-------|
| `staticCounter` (value) | ? | |
| `instanceField` inside `obj` | ? | |
| `localPrimitive` | ? | |
| `obj` reference in main | ? | |
| `MemoryTrace` object | ? | |
| `param` in process | ? | |
| `ref` in process | ? | |
| `localInMethod` | ? | |

<details>
<summary>Answers</summary>

| Variable / object | Location |
|-------------------|----------|
| `staticCounter` value | Metaspace/static storage (conceptually with class); not on stack |
| `instanceField` | **Heap** (inside object) |
| `localPrimitive` | **Stack** (main's frame) |
| `obj` reference | **Stack** (main's frame) |
| `MemoryTrace` object | **Heap** |
| `param` | **Stack** (process's frame) |
| `ref` | **Stack** (reference); points to heap object |
| `localInMethod` | **Stack** |

</details>

---

### Exercise 2: Thread confinement or shared?

```java
public class Confinement {
    void threadSafe() {
        StringBuilder sb = new StringBuilder();  // A
        sb.append("hello");
    }

    private StringBuilder shared = new StringBuilder();  // B

    void notThreadSafe() {
        shared.append("x");  // C
    }
}
```

| Line | Thread-safe if called from multiple threads? | Why |
|------|---------------------------------------------|-----|
| A–C in `threadSafe()` | ? | |
| C in `notThreadSafe()` | ? | |

<details>
<summary>Answers</summary>

- **`threadSafe()`** — YES. `sb` is local (stack ref); never escapes → only one thread touches the heap `StringBuilder`.
- **`notThreadSafe()`** — NO. `shared` is heap field; multiple threads mutate same `StringBuilder` → race on internal array/count.

</details>

---

### Exercise 3: Reproduce StackOverflowError

```java
public class StackDemo {
    static void recurse(int depth) {
        recurse(depth + 1);
    }

    public static void main(String[] args) {
        try {
            recurse(0);
        } catch (StackOverflowError e) {
            System.out.println("Stack overflow caught");
        }
    }
}
```

**Apply:**
1. Run as-is — observe `StackOverflowError`.
2. Run with `java -Xss2m StackDemo` — notice deeper recursion before overflow.
3. **Handwritten note:** stack stores frames; each call adds frame; `-Xss` increases limit.

---

### Exercise 4: Reproduce heap OutOfMemoryError (lab only)

```java
import java.util.*;

public class HeapDemo {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        try {
            while (true) {
                list.add(new byte[1024 * 1024]); // 1 MB each
            }
        } catch (OutOfMemoryError e) {
            System.out.println("Heap OOM after " + list.size() + " MB");
        }
    }
}
```

**Run:** `java -Xmx32m HeapDemo`  
**Handwritten note:** objects on heap; list holds references → GC cannot collect → OOM.

---

## Part B — JMM Visibility Exercises

### Exercise 5: Broken stop flag — predict behavior

```java
public class BrokenStop {
    private boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        BrokenStop w = new BrokenStop();
        Thread t = new Thread(w::runLoop, "worker");
        t.start();
        Thread.sleep(100);
        w.stop();
        t.join(2000);
        System.out.println("Thread alive? " + t.isAlive());
    }

    void runLoop() {
        while (running) { /* busy */ }
        System.out.println("Stopped cleanly");
    }

    void stop() { running = false; }
}
```

**Questions:**
1. Might worker loop forever? **Yes** — no happens-before between `stop()` write and `runLoop()` read.
2. Fix with one keyword? **`volatile`** on `running`.

**Fixed:**

```java
private volatile boolean running = true;
```

**happens-before chain after fix:**  
`running = false` (volatile write) **hb** `while (running)` read → worker sees false.

---

### Exercise 6: volatile is not enough for count++

```java
public class VolatileCounter {
    private volatile int count = 0;

    void increment() { count++; }

    public static void main(String[] args) throws InterruptedException {
        VolatileCounter c = new VolatileCounter();
        Thread t1 = new Thread(() -> { for (int i = 0; i < 100_000; i++) c.increment(); });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 100_000; i++) c.increment(); });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("count = " + c.count);  // often < 200_000
    }
}
```

**Apply:**
1. Run — observe count < 200_000.
2. Explain: `count++` = read, add, write — not atomic; volatile only makes each individual read/write visible, not the whole operation.

**Fix options:**

```java
// Option A: synchronized
synchronized void increment() { count++; }

// Option B: AtomicInteger
private final AtomicInteger count = new AtomicInteger();
void increment() { count.incrementAndGet(); }
```

---

### Exercise 7: happens-before with thread start/join

```java
public class StartJoinHB {
    static int result = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> result = 42);
        t.start();
        t.join();
        System.out.println(result);  // always 42
    }
}
```

**Trace happens-before:**
1. `result = 42` in worker **hb** `t.join()` returning (thread join rule).
2. Main reads `result` after join → guaranteed to see 42.

**Broken variant** — remove `join()`:

```java
t.start();
System.out.println(result);  // may print 0 — no hb
```

---

### Exercise 8: synchronized visibility

```java
public class SyncVisibility {
    private int x = 0;
    private int y = 0;
    private final Object lock = new Object();

    void writer() {
        synchronized (lock) {
            x = 1;
            y = 2;
        }
    }

    int reader() {
        synchronized (lock) {
            return x + y;  // guaranteed 3 if writer finished unlock before this lock
        }
    }
}
```

**Apply:** Unlock after writer **hb** lock in reader → reader sees both writes.

---

## Part C — Combined Stack/Heap + JMM Scenarios

### Scenario 1: Local cache of shared object

```java
public class LocalRefSharedObject {
    private final List<String> items = new ArrayList<>();

    void add(String s) {
        List<String> local = items;  // local ref on STACK
        local.add(s);                // mutates SHARED heap list
    }
}
```

| Question | Answer |
|----------|--------|
| Is `local` thread-safe? | Ref is on stack, but **points to shared heap list** — not safe multi-threaded |
| Fix? | `synchronized (items) { items.add(s); }` or `Collections.synchronizedList` / `CopyOnWriteArrayList` |

---

### Scenario 2: Returning escaped reference

```java
public class EscapingReference {
    private int[] data;

    int[] getData() {
        int[] local = new int[100];  // heap array; local ref on stack
        data = local;                // ESCAPE — stored in instance field
        return local;                // double escape
    }
}
```

**Apply:**
- Array allocated on **heap**.
- `local` ref on **stack** during method.
- Assignment to `data` publishes to other threads → needs JMM consideration if read without sync.

---

### Scenario 3: Immutable + safe publication

```java
public class ImmutableConfig {
    private final Map<String, String> config;

    ImmutableConfig(Map<String, String> input) {
        this.config = Map.copyOf(input);  // immutable copy
    }

    Map<String, String> getConfig() {
        return config;  // safe to share — no mutation possible
    }
}
```

**Apply:** `final` field + immutable object = safe publication without locks (readers see fully constructed map).

---

## Part D — Code Fix Drills (exam-style)

Fix each snippet in one sentence, then implement.

### Drill 1

```java
class Status {
    int code = 0;
    void set(int c) { code = c; }
    int get() { return code; }
}
// Used from many threads
```

**Fix:** Use `AtomicInteger` or synchronize both methods / use `volatile` if only single writes and reads (not compound).

---

### Drill 2

```java
class LazyInit {
    private Helper helper;
    Helper getHelper() {
        if (helper == null)
            helper = new Helper();
        return helper;
    }
}
```

**Fix:** Synchronize, use holder idiom, or `volatile` + DCL correctly.

---

### Drill 3

```java
void badRecursion() {
    badRecursion();
}
```

**Fix:** Base case; this is stack exhaustion, not JMM.

---

## Part E — Self-Test Checklist

After practice, you should be able to:

- [ ] Draw JVM memory diagram with heap, metaspace, per-thread stacks
- [ ] For any variable, say stack vs heap vs metaspace
- [ ] Explain why local variables are usually thread-safe
- [ ] State 5 happens-before rules from memory
- [ ] Fix a non-volatile stop flag
- [ ] Explain why volatile doesn't fix `count++`
- [ ] Choose between volatile, synchronized, and AtomicInteger
- [ ] Reproduce StackOverflowError and explain `-Xss`
- [ ] Explain heap OOM vs metaspace OOM
- [ ] Trace class loading phases Load → Link → Init

---

## Part F — Quick Reference Card (practice outputs)

### Decision flowchart (draw by hand)

```
Variable declared inside method?
  YES → reference/value on STACK (frame)
        object pointed to → HEAP
  NO → static? → metaspace/static area
       instance field? → inside object on HEAP

Multiple threads access same mutable heap data?
  YES → need happens-before:
        volatile (single read/write visibility)
        synchronized (visibility + exclusion)
        java.util.concurrent.atomic.* (CAS + visibility)
  NO → thread confinement → safe
```

### happens-before quick drill

Fill blanks:

1. Unlock **hb** subsequent ______ on same monitor  
2. Volatile write **hb** subsequent ______ of same variable  
3. `start()` **hb** first action in ______  
4. Last action in thread **hb** ______ returning  

<details>
<summary>Answers</summary>

1. lock  
2. read  
3. new thread  
4. join  

</details>

---

## Part G — Optional Hands-On Commands

```bash
# See class loading
java -verbose:class MyApp 2>&1 | head -20

# GC activity while running HeapDemo
java -Xmx32m -verbose:gc HeapDemo

# Disassemble to connect bytecode → stack operations
javac MemoryTrace.java && javap -c MemoryTrace

# Thread dump if practicing concurrency bugs
jstack <pid>
```

---

## Handwritten Cheat Sheet (practice summary)

```
TRACE: local? → stack | new? → heap | static? → metaspace
       ref on stack ≠ object on stack

CONFINEMENT: local var, no escape → no sync needed
ESCAPE: assign to field / return / store static → shared → sync

VISIBILITY FIX: volatile (flag), synchronized (block), start/join
ATOMICITY FIX: synchronized or AtomicInteger (NOT volatile alone)

count++ shared = BROKEN (read-modify-write)
stop flag without volatile = may loop forever

StackOverflow = recursion, -Xss increases depth
Heap OOM = too many live objects, -Xmx

Practice pattern:
  1. Where is data?
  2. Who can see it?
  3. Is there happens-before?
  4. Need atomicity or only visibility?
```

---

**Next note:** #014 — Weekly review + spaced repetition (Anki); fix gaps
