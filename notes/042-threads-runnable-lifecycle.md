# #042 — Threads, Runnable, Thread Lifecycle

**Date:** 2026-10-12  
**Roadmap row:** 42 — P1 Java Core · Concurrency & Multithreading I · *Learn*  
**Sources:** `java.lang.Thread` Javadoc (JDK 21), *Java Concurrency in Practice* (Goetz et al.) ch. 1, JEP 444 (virtual threads)

> Every ✔︎ result was produced by running the demo on JDK 21.0.10, on an Apple M4 (10 cores: 4 performance + 6 efficiency).

---

## At a Glance (copy to handwritten notes)

- A **thread** is an independent path of execution with its **own stack** (and PC register), sharing the process's **heap** with all the other threads (#004).
- Define **what** to run with a `Runnable` (or `Callable`). Run it by calling **`start()`**, which creates a new OS thread. Calling **`run()` just executes the method in the *current* thread** ✔︎.
- `start()` can be called **once**; a second call throws `IllegalThreadStateException` ✔︎.
- **Six states** (`Thread.State`) ✔︎: `NEW` → `RUNNABLE` ⇄ `BLOCKED` (waiting for a monitor) / `WAITING` (`wait()`, `join()`, `park()`) / `TIMED_WAITING` (`sleep(ms)`, `wait(ms)`) → `TERMINATED`.
- **`join()`** waits for a thread to finish, and creates a **happens-before** edge: everything the thread did is visible after `join` returns (#012).
- **Interruption is cooperative:** `interrupt()` sets a flag. Blocking methods throw `InterruptedException` **and clear the flag** ✔︎, so either propagate the exception or **restore the flag** with `Thread.currentThread().interrupt()`.
- An exception in a thread kills **only that thread**; the uncaught-exception handler sees it ✔︎. It does **not** reach the thread that called `start()`.
- **Daemon** threads don't keep the JVM alive; at exit they're abandoned, and **`finally` blocks don't run** ✔︎.
- **Virtual threads (Java 21)** are cheap threads scheduled by the JVM onto a few carrier threads, ideal for blocking I/O. ✔︎ 10 000 tasks × `sleep(100 ms)`: **10.4 s** on a 100-thread pool vs **147 ms** on virtual threads.

---

## 1. Process vs Thread

| | Process (a JVM) | Thread |
|---|---|---|
| Memory | Own address space | **Shares the heap** and Metaspace with the other threads of the process |
| Private to it | Everything | **JVM stack** (frames, locals), **PC register**, native stack (#004) |
| Creation cost | High | A platform thread costs ~1 MB of reserved stack plus an OS thread; a virtual thread costs a few hundred bytes to KBs |
| Communication | IPC (sockets, pipes, files) | **Shared memory**: fast, but it needs synchronisation (rows 45, 48) |
| One crashes | Others are unaffected | An uncaught exception ends only *that* thread; the JVM continues |

**Why use threads at all** (JCIP ch. 1): exploit multiple cores, keep things responsive (for example, don't block a UI or a request loop while waiting on I/O), and model naturally concurrent activities (one handler per request).

**What they cost** (JCIP ch. 1): **safety hazards** (race conditions, rows 49 and 51), **liveness hazards** (deadlock, starvation, rows 54–55) and **performance hazards** (context switches, lost cache locality, synchronisation overhead).

---

## 2. Creating Threads

```java
// 1. Runnable as a lambda (preferred: separates the task from the mechanism)
Thread t1 = new Thread(() -> System.out.println("hi"), "worker-1");

// 2. A Runnable class
class Download implements Runnable {
    public void run() { /* ... */ }
}
Thread t2 = new Thread(new Download(), "download");

// 3. Subclassing Thread (avoid: ties the task to one execution mechanism, and uses up your single superclass)
class MyThread extends Thread {
    @Override public void run() { /* ... */ }
}

// 4. Java 21 builders
Thread p = Thread.ofPlatform().name("io-", 0).start(task);      // named io-0, io-1, ...
Thread v = Thread.ofVirtual().start(task);                       // virtual thread

// 5. In real code: an executor (rows 67+); you submit tasks, it manages the threads
try (ExecutorService ex = Executors.newFixedThreadPool(4)) {
    Future<Integer> f = ex.submit(() -> 42);                     // Callable: returns a value, can throw
}
```

| | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `void run()` | `V call() throws Exception` |
| Returns a value | ✗ | ✅ (via a `Future<V>`) |
| Checked exceptions | ✗ | ✅ |
| Used with | `new Thread(r)`, `executor.execute(r)` | `executor.submit(c)` |

**Prefer `Runnable`/`Callable` + an executor over subclassing `Thread`.** It's composition over inheritance again (EJ Item 18, #003): the task doesn't care how it's run.

### `start()` vs `run()` ✔︎

```java
Thread t = new Thread(() -> System.out.println("task runs on thread: " + Thread.currentThread().getName()), "worker");
t.run();    // ✔︎ "task runs on thread: main"    ← an ordinary method call; NO new thread
t.start();  // ✔︎ "task runs on thread: worker"  ← a new thread executes run()
t.start();  // ✔︎ IllegalThreadStateException     ← a thread can only be started once
```

Calling `run()` directly is a classic bug: everything works, just **sequentially**, and nothing tells you.

---

## 3. The Thread Lifecycle (six states) ✔︎

```
                 start()
        NEW ──────────────────▶ RUNNABLE ─────────── run() returns / throws ──────────▶ TERMINATED
                               (running, or ready
                                to run; includes
                                blocking I/O)
                           ▲   │   ▲   │   ▲   │
       monitor acquired    │   │   │   │   │   │  sleep(ms), wait(ms), join(ms),
                           │   ▼   │   ▼   │   ▼  parkNanos
                         BLOCKED  WAITING  TIMED_WAITING
           (waiting to enter a     (wait(), join(),     (time-limited versions;
            synchronized block)     LockSupport.park())   they return on timeout too)
```

Observed by `getState()` in the demo:

| Situation | State ✔︎ |
|-----------|---------|
| `new Thread(...)`, not started | `NEW` |
| Inside `Thread.sleep(500)` | `TIMED_WAITING` |
| A busy loop | `RUNNABLE` |
| Trying to enter `synchronized (lock)` while another thread holds `lock` | `BLOCKED` |
| Inside `lock.wait()` | `WAITING` |
| Inside `LockSupport.park()` (what `ReentrantLock`, `BlockingQueue`… use underneath) | `WAITING` |
| After `run()` returned | `TERMINATED` |

Notes:

- **`RUNNABLE` doesn't mean running on a CPU.** It means "eligible to run". A thread blocked in **socket I/O** is also reported as `RUNNABLE` by the JVM.
- **`BLOCKED`** is specific to waiting for an **intrinsic lock** (`synchronized`). Waiting on a `ReentrantLock` shows as `WAITING`, because it parks.
- Thread dumps (`jstack <pid>`, `jcmd <pid> Thread.print`) show these states. They're the first tool for diagnosing hangs and deadlocks (rows 54–55).

---

## 4. Waiting for Threads: `join`

```java
Thread worker = new Thread(() -> result[0] = compute());
worker.start();
worker.join();             // the current thread WAITS until worker is TERMINATED
use(result[0]);            // safe: join() establishes happens-before (#012)
```

- `join(millis)` waits at most that long; check `isAlive()` afterwards to find out whether it finished.
- **Visibility guarantee:** all writes by a thread *happen-before* another thread's successful return from `join()` on it. Without `join` (or another happens-before edge), reading `result[0]` from the main thread is a **data race** (#012 §4).

---

## 5. Sleeping, Yielding, Priorities

| Method | Effect | Releases locks it holds? |
|--------|--------|:---:|
| `Thread.sleep(ms)` | → `TIMED_WAITING` for at least `ms` | **No** |
| `Thread.yield()` | A *hint* that the thread is willing to give up the CPU; often a no-op | No |
| `obj.wait()` (row 46) | Waits on `obj`'s monitor | **Yes**, releases `obj`'s monitor |
| `setPriority(1..10)` | A *hint* to the OS scheduler (default 5 ✔︎); not portable | — |

**Never use `sleep` for coordination** ("sleep 100 ms and hope the other thread has finished"). Use `join`, latches, queues or futures. `sleep` in tests is the #1 source of flaky concurrency tests.

---

## 6. Interruption: Cooperative Cancellation

Java has **no safe way to forcibly stop a thread** (`Thread.stop()` is deprecated and, since Java 20, throws `UnsupportedOperationException`). Instead:

```java
worker.interrupt();        // ask: set worker's interrupt flag
```

The target thread must cooperate:

| Where the target is | What `interrupt()` does |
|---------------------|-------------------------|
| Blocked in `sleep`, `wait`, `join`, `BlockingQueue.take`… | The method **throws `InterruptedException`** and **clears the flag** ✔︎ |
| Running ordinary code | Only sets the flag; the code must poll `Thread.currentThread().isInterrupted()` |
| Blocked in classic `java.io` I/O | **Nothing**; it stays blocked (NIO's interruptible channels are different) |

✔︎ Demo:

```text
caught InterruptedException; flag now = false     ← the exception cleared the flag
after re-interrupt, flag = true                   ← Thread.currentThread().interrupt() restores it
Thread.interrupted() = true, then again = false   ← the STATIC method tests AND clears
```

### The two correct ways to handle `InterruptedException`

```java
// 1. Propagate it (best: let the caller decide)
void process() throws InterruptedException {
    Item item = queue.take();
}

// 2. If you can't throw it (e.g. inside Runnable.run): restore the flag, then wind down
public void run() {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            Item item = queue.take();
            handle(item);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();   // preserve the evidence for code higher up
    }
}

// ❌ NEVER swallow it silently:
try { Thread.sleep(1000); } catch (InterruptedException e) { }   // the cancellation request is lost
```

`isInterrupted()` (an instance method) only **reads** the flag. `Thread.interrupted()` (static) reads **and clears** the current thread's flag ✔︎. Mixing them up is a common bug.

---

## 7. Exceptions in Threads ✔︎

```java
Thread boom = new Thread(() -> { throw new IllegalStateException("boom"); }, "boom");
boom.setUncaughtExceptionHandler((th, ex) -> log.error("{} died", th.getName(), ex));
boom.start();
boom.join();   // ✔︎ the handler gets "boom"; main keeps running normally
```

- The exception **doesn't propagate** to the thread that called `start()`. Without a handler, the default one prints the stack trace to `System.err`, and the thread just dies.
- `Thread.setDefaultUncaughtExceptionHandler(...)` sets a JVM-wide fallback (useful for logging).
- With an executor, `submit()` captures the exception inside the `Future` (you only see it when you call `future.get()`), while `execute()` reaches the uncaught handler. A classic "my task failed silently" trap (rows 67+).

---

## 8. Daemon Threads ✔︎

```text
$ java Daemon.java            (worker NOT daemon, sleeping 2 s)
main exiting (worker daemon=false)
worker finished normally
worker finally ran            ← the JVM waited for it

$ java Daemon.java daemon     (worker IS daemon)
main exiting (worker daemon=true)
                              ← the JVM exited immediately; the worker was abandoned, finally NEVER ran
```

- The JVM exits when only **daemon** threads remain. Daemon threads are abandoned mid-execution, with no `finally` and no flushing.
- New threads **inherit** the daemon flag of the thread that creates them (main is non-daemon ✔︎). Set it **before** `start()`.
- Use daemons for housekeeping that's safe to kill (cache cleanup, metrics). **Never** for work that must complete, such as writing files or sending messages.
- **Virtual threads are always daemon threads** ✔︎ (`isDaemon=true`).

---

## 9. Virtual Threads (Java 21, JEP 444) ✔︎

**Platform thread** = a thin wrapper around an **OS thread** (expensive, typically thousands at most). **Virtual thread** = a JVM-managed thread that is **mounted on a carrier** (a platform thread from a small ForkJoinPool, one per core by default) only while it's running. When it **blocks** (sleep, socket I/O, `BlockingQueue.take`), it's **unmounted**, freeing the carrier for another virtual thread.

✔︎ **Measured:** 10 000 tasks, each doing `Thread.sleep(100)`:

| Executor | Wall time | Why |
|----------|----------:|-----|
| `newFixedThreadPool(100)` | **10 421 ms** | 100 threads × 100 batches × 100 ms ≈ 10 s |
| `newVirtualThreadPerTaskExecutor()` | **147 ms** | All 10 000 sleep *concurrently*, with no OS thread per task |

| Use virtual threads for | Don't expect them to help with |
|-------------------------|--------------------------------|
| Many concurrent **blocking** tasks: HTTP calls, DB queries, thread-per-request servers | **CPU-bound** work (still limited by core count; use a fixed pool sized to the cores) |
| Simple blocking code instead of async callbacks | Pooling them (they're cheap: create one per task, don't pool) |

Caveat in JDK 21: blocking **inside `synchronized`** **pins** the virtual thread to its carrier (it can't unmount), which can starve the carriers. Prefer `ReentrantLock` around blocking calls in JDK 21. (JEP 491 removes most pinning in **JDK 24**.) More in row 45.

---

## 10. Thread API Quick Reference

| Method | Purpose |
|--------|---------|
| `start()` / `run()` | Start a new thread / just call the method (no new thread) |
| `join()`, `join(ms)` | Wait for termination (happens-before) |
| `sleep(ms)` (static) | Pause the current thread; keeps its locks |
| `interrupt()` / `isInterrupted()` / `Thread.interrupted()` | Request cancellation / read the flag / read and clear (static) |
| `getState()`, `isAlive()` | Introspection |
| `setDaemon(b)` | Must be called before `start()` |
| `setName`, `getName`, `threadId()` (19+) | Identity (`getId()` is deprecated) |
| `setUncaughtExceptionHandler` | Per-thread handler for exceptions |
| `Thread.currentThread()` | The thread running this code |
| `Thread.ofVirtual()`, `Thread.ofPlatform()`, `Thread.startVirtualThread(r)` | Java 21 builders |
| ~~`stop()`, `suspend()`, `resume()`~~ | Deprecated and unsafe; throw `UnsupportedOperationException` in modern JDKs |

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| `start()` vs `run()`? | `start()` creates a thread that calls `run()`; calling `run()` executes in the current thread |
| Can you restart a thread? | No: `IllegalThreadStateException`. Create a new one, or use an executor |
| Runnable vs Callable? | Callable returns a value and can throw checked exceptions |
| Name the thread states | NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED |
| BLOCKED vs WAITING? | BLOCKED: waiting to **acquire** a monitor. WAITING: voluntarily waiting (`wait`/`join`/`park`) to be woken |
| Does `sleep` release locks? | No. `wait` releases the monitor it's called on |
| How do you stop a thread? | `interrupt()` + cooperative checks; never `stop()` |
| What do you do with `InterruptedException`? | Propagate it, or restore the flag with `Thread.currentThread().interrupt()`. Never swallow it |
| `interrupted()` vs `isInterrupted()`? | Static read-and-clear vs instance read-only |
| What happens to an exception in a thread? | It ends that thread; the uncaught handler sees it; the starter doesn't |
| Daemon threads? | They don't keep the JVM alive; they're abandoned at exit with no `finally` |
| Virtual threads: when? | Lots of blocking tasks; not for CPU-bound work; don't pool them |

---

## Handwritten Cheat Sheet

```
THREAD = own stack + PC; SHARED heap
CREATE: new Thread(runnable, "name") · Thread.ofVirtual().start(r) · executor.submit(callable)
start() = new thread · run() = same thread (silent bug) · start twice → IllegalThreadStateException
STATES: NEW → RUNNABLE ⇄ BLOCKED (monitor) / WAITING (wait, join, park) / TIMED_WAITING (sleep, wait(ms)) → TERMINATED
join() = wait + happens-before
sleep keeps locks · wait releases the monitor
INTERRUPT = cooperative flag; blocking calls throw InterruptedException AND CLEAR the flag
  → propagate, or Thread.currentThread().interrupt(); never swallow
  isInterrupted() reads · Thread.interrupted() reads + clears
EXCEPTION in a thread → only it dies → UncaughtExceptionHandler
DAEMON: JVM exits without waiting; no finally; set before start; virtual threads are daemon
VIRTUAL (21): cheap, unmount on blocking; 10k×sleep(100): pool(100) 10.4 s vs virtual 0.15 s
  for blocking I/O, not CPU work; don't pool; synchronized+blocking pins (JDK 21)
```

---

**Next:** row 43 — Create & coordinate threads (#043)
