# #044 — Read: *Java Concurrency in Practice*, Chapters 1–4 — Reading Notes

**Date:** 2026-10-12  
**Roadmap row:** 44 — P1 Java Core · Concurrency & Multithreading I · *Flex*  
**Source:** Brian Goetz, Tim Peierls, Joshua Bloch, Joseph Bowbeer, David Holmes, Doug Lea, *Java Concurrency in Practice* (Addison-Wesley, 2006): Ch. 1 *Introduction*, Ch. 2 *Thread Safety*, Ch. 3 *Sharing Objects*, Ch. 4 *Composing Objects*

> JCIP predates Java 8 (no lambdas, `LongAdder`, `CompletableFuture` or virtual threads), but Chapters 2–4 are about **principles** that haven't changed. Where this repo reproduced one of the book's examples on JDK 21, it's marked ✔︎ with a link to the row.

---

## At a Glance (copy to handwritten notes)

- **Ch. 1:** threads bring **safety** hazards (nothing bad happens), **liveness** hazards (something good eventually happens) and **performance** hazards.
- **Ch. 2:** *thread safety = correct behaviour when accessed from multiple threads, with no extra synchronisation by callers.* Races come in two shapes: **check-then-act** and **read-modify-write**. **Stateless objects are always thread-safe.** Guard **every** access to a state variable with the **same** lock; **intrinsic locks are reentrant**; keep critical sections short.
- **Ch. 3:** **visibility** matters as much as atomicity. Without synchronisation you may see **stale data** (or none at all). Use **`volatile`** only for simple flags or publication. **Don't let `this` escape** during construction. Avoid sharing with **thread confinement** (stack, `ThreadLocal`); make things **immutable**; publish **safely**.
- **Ch. 4:** design a thread-safe class by identifying its **state variables**, **invariants** and **synchronisation policy**. Use **instance confinement** and the **Java monitor pattern**, **delegate** to thread-safe components (only if the state variables are independent), and **document** the policy. **Client-side locking must use the same lock** the class uses ✔︎.

---

## Chapter 1 — Introduction

### Why threads (benefits)

1. **Exploiting multiple processors.** One thread uses one core; a 10-core machine idles 90% (#043 measured 4.9× on 10 cores).
2. **Simpler modelling.** One thread per independent task (a request, a client) can be easier than a hand-built event loop.
3. **Simplified handling of asynchronous events.** Blocking I/O in its own thread doesn't stall everything else.
4. **Responsive user interfaces.** Long tasks run off the UI/event thread.

### Risks

| Hazard | Meaning | Example in this repo |
|--------|---------|----------------------|
| **Safety** | "Nothing bad ever happens": correctness | `UnsafeSequence`: `value++` loses updates ✔︎ (#049: lost 76%) |
| **Liveness** | "Something good eventually happens": no deadlock, starvation or livelock | Lost wake-up with `notify` → **HANG in 50/50 runs** ✔︎ (#046) |
| **Performance** | Context switches, lost locality, synchronisation overhead | `AtomicLong` at 8 threads drops to 7% of its single-thread throughput ✔︎ (#050) |

**Threads are everywhere**, even if you never call `new Thread`: Timers, servlets/Spring request handlers, RMI, and Swing/JavaFX callbacks all run your code on threads you didn't create, so **your** objects must be thread-safe when frameworks call them concurrently.

---

## Chapter 2 — Thread Safety

### 2.1 What is thread safety?

> *"A class is thread-safe if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization or other coordination on the part of the calling code."*

- "Correctly" means it satisfies its **specification**: invariants (constraints on state) and postconditions.
- **Stateless objects are always thread-safe** (`StatelessFactorizer`): everything lives in local variables on each thread's stack. That's why stateless services (Spring singletons with no mutable fields) are safe to share.

### 2.2 Atomicity

- **Race condition** = the correctness of a computation depends on the relative timing of threads.
- **Check-then-act**: `if (instance == null) instance = new …` (`LazyInitRace`). Two threads both see `null` and both create.
- **Read-modify-write**: `count++` (`UnsafeCountingFactorizer`).
- **Compound actions** must execute **atomically** (all or nothing, from other threads' point of view).
- For a single variable, use **`java.util.concurrent.atomic`** classes (`AtomicLong`): `CountingFactorizer`.

✔︎ Reproduced in #049: check-then-act seat booking sold **4 651 tickets for 1 000 seats**; `count++` lost 1.5 M of 2 M.

### 2.3 Locking

> *"To preserve state consistency, update related state variables in a single atomic operation."*

`UnsafeCachingFactorizer` stores `lastNumber` and `lastFactors` in two `AtomicReference`s. Each is individually atomic, but the **pair** must change together (an invariant across variables), so two atomics aren't enough.

- **Intrinsic locks** (`synchronized`) act as mutexes. Synchronising the whole `service()` method fixes it (`SynchronizedFactorizer`) but serialises every request, which is **poor concurrency**.
- **Reentrancy:** locks are acquired per **thread**, not per invocation, so a subclass's `synchronized` method can call `super`'s `synchronized` method ✔︎ (#045 §4).

### 2.4 Guarding state with locks

> *"For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock."*

> *"For every invariant that involves more than one variable, all the variables involved in that invariant must be guarded by the same lock."*

- Making every method `synchronized` doesn't make **compound** actions on the object atomic (`if (!vector.contains(e)) vector.add(e)` is still a race).
- Document the policy: JCIP's `@GuardedBy("lock")` annotation (or a comment) on each field.

### 2.5 Liveness and performance

`CachedFactorizer`: **narrow the synchronized blocks** to the parts that touch shared state, and do the expensive factorisation **outside** the lock.

> *"Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network or console I/O."*

Balance simplicity (big blocks) against concurrency (small blocks), and **don't prematurely split** synchronisation for performance.

---

## Chapter 3 — Sharing Objects

### 3.1 Visibility

`NoVisibility`: a reader thread spins on a non-volatile `ready` flag. It may **loop forever** or print **0** (seeing `ready` before `number`).

✔︎ Reproduced in #048: a spin loop on a plain flag **never stopped**; it stopped under `-Xint`; with `volatile` it stopped in 0.1 ms. And a store-buffering litmus test produced the "impossible" both-zero result in 22–30% of rounds.

- **Stale data:** a reader can see an out-of-date value, or a mix of up-to-date and stale fields. `MutableInteger` with an unsynchronised getter is broken; `SynchronizedInteger` synchronises **both** `get` and `set`.
- **Non-atomic 64-bit operations:** even stale reads normally return a value some thread actually wrote (JCIP calls this *out-of-thin-air safety*). Non-volatile `long`/`double` are the exception: the JLS lets the JVM write them as two 32-bit halves, so a reader could see half of one write and half of another. Use `volatile` or a lock.
- **Locking gives visibility:** everything A did before releasing lock M is visible to B after B acquires M (#045 §5).
- **Volatile variables:** visibility and ordering, with no locking. *"Use volatile variables only when they simplify implementing and verifying your synchronization policy."* The criteria (#048 §4): writes don't depend on the current value (or a single writer), not part of an invariant, and no other reason to lock. The canonical use is a status flag (`asleep`).

### 3.2 Publication and escape

- **Publishing** an object makes it available outside its current scope. **Escape** is publishing it when it shouldn't be, or before it's ready.
- Returning a reference to an internal mutable array (`UnsafeStates.getStates()`) publishes internal state.
- **`ThisEscape`:** registering a listener (an inner class instance) inside the constructor publishes `this` before construction finishes, so the listener can see a half-built object.
- **Fix:** a private constructor + a factory method (`SafeListener.newInstance`) that registers the listener **after** construction.

> *"Do not allow the `this` reference to escape during construction."*

(Java 21's `-Xlint:this-escape` flags some of these at compile time. It caught exactly this in the #025 `BetterHashMap` copy constructor.)

### 3.3 Thread confinement

If data is only ever accessed from **one** thread, it needs no synchronisation.

| Kind | How |
|------|-----|
| **Ad-hoc** | By convention only (fragile) |
| **Stack confinement** | Local variables (and objects only reachable from them) live on one thread's stack ✔︎ (#043: each worker counts in a local, publishing once) |
| **`ThreadLocal`** | A per-thread copy (e.g. a per-thread `SimpleDateFormat`, or JDBC connections in some frameworks). Watch out for leaks in thread pools (values outlive the task) |

Swing's single event-dispatch thread and JDBC `Connection` pooling are confinement by design.

### 3.4 Immutability

> *"Immutable objects are always thread-safe."*

An object is immutable if its state can't change after construction, all its fields are `final`, and it's properly constructed (`this` didn't escape).

- `final` fields have special **initialisation-safety** guarantees: other threads see the correct values of final fields (and of what they reference, as of construction) **without synchronisation**.
- **Immutable holder + volatile reference:** `VolatileCachedFactorizer` puts `lastNumber` and `lastFactors` in an immutable `OneValueCache` and swaps a `volatile` reference to it. The multi-variable invariant becomes **one atomic reference swap** (#048 §4).

> *"Just as it is a good practice to make all fields private unless they need greater visibility, it is a good practice to make all fields final unless they need to be mutable."*

### 3.5 Safe publication

`Holder` with `assertSanity()` (`if (n != n) throw …`) can **fail** when published without synchronisation: a thread can see a partially constructed object.

**Safe publication idioms** (#048 §8):

1. A static initialiser.
2. A `volatile` field or `AtomicReference`.
3. A `final` field of a properly constructed object.
4. A field guarded by a lock.
5. A thread-safe collection (`Hashtable`, `ConcurrentMap`, `synchronizedList`, `BlockingQueue`…).

| Object kind | Requirement |
|-------------|-------------|
| **Immutable** | Can be published through **any** mechanism |
| **Effectively immutable** (never modified after publication) | Must be **safely published** |
| **Mutable** | Must be safely published **and** either thread-safe or guarded by a lock |

**Policies for sharing:** thread-confined · shared read-only · shared thread-safe (internally synchronised) · guarded (accessed only with a specific lock held).

---

## Chapter 4 — Composing Objects

### 4.1 Designing a thread-safe class

1. Identify the **variables** that form the object's state.
2. Identify the **invariants** that constrain them (e.g. `lower ≤ upper`).
3. Establish a **policy** for managing concurrent access (which lock guards what).

Also: **state-dependent operations** (preconditions such as "the buffer is not empty", #046) and **state ownership** (who owns an object passed in, and who may mutate it).

### 4.2 Instance confinement

Encapsulate non-thread-safe state inside an object and only access it while holding that object's lock (`PersonSet` wraps a `HashSet` and synchronises every method).

**The Java monitor pattern:** all mutable state is encapsulated and guarded by the object's own intrinsic lock (`Vector`, `Hashtable`). Using a **private lock object** instead is equivalent but better encapsulated, since clients can't acquire it (#045 §7).

`MonitorVehicleTracker` returns **deep copies** of its mutable `MutablePoint`s, so callers never touch internal state. That's correct, but copying gets expensive.

### 4.3 Delegating thread safety

`DelegatingVehicleTracker` stores **immutable** `Point`s in a `ConcurrentHashMap` and has no locks of its own; the thread-safe map handles it.

**Delegation works when the state variables are independent.** `NumberRange` has two `AtomicInteger`s for `lower`/`upper`, each thread-safe, but the invariant `lower ≤ upper` can be violated by a check-then-act in `setLower`/`setUpper`. So **delegation fails when there are invariants between variables**.

> *"If a class is composed of multiple independent thread-safe state variables and has no operations that have any invalid state transitions, then it can delegate thread safety to the underlying state variables."*

Publishing the underlying state is safe only if it's thread-safe, isn't constrained by invariants, and has no prohibited state transitions (`SafePoint`, `PublishingVehicleTracker`).

### 4.4 Adding functionality to existing thread-safe classes

To add `putIfAbsent` to a synchronised list, there are four options:

| Approach | Verdict |
|----------|---------|
| **Modify the class** | Best, if you own it |
| **Extend it** (`BetterVector extends Vector`) | Fragile: it depends on the base class's synchronisation policy staying the same |
| **Client-side locking** | Must lock **the same lock the collection uses**. Fragile, because it spreads the synchronisation policy around |
| **Composition** (`ImprovedList` wraps a `List` and synchronises on itself, delegating every method) | ✅ the most robust: it has its own consistent lock |

**`ListHelper`, the broken version:**

```java
public class ListHelper<E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
    public synchronized boolean putIfAbsent(E x) {     // locks the HELPER
        boolean absent = !list.contains(x);
        if (absent) list.add(x);
        return absent;
    }
}
```

`synchronized` here locks the **helper**, but other code operates on `list`, whose methods lock the **list**. They're different locks, so `putIfAbsent` isn't atomic with respect to other list operations. The fix is `synchronized (list) { … }`.

✔︎ **Reproduced** (8 threads: 4 using the helper, 4 doing a correctly client-side-locked put-if-absent on the list, 2 000 values, 20 runs):

```text
Broken helper (synchronized on this): duplicates in 4/20 runs, worst 1
Good helper (synchronized on list):   duplicates in 0/20 runs
```

The broken version fails **rarely** (1 duplicate in 4 of 20 runs). That's exactly the kind of bug that survives code review and testing. (A first attempt at this experiment, where the second group of threads did *unsynchronised* check-then-add, showed far more duplicates, but that measured a *different* race. Isolating the variable under test matters.)

### 4.5 Documenting synchronisation policies

> *"Document a class's thread safety guarantees for its clients; document its synchronization policy for its maintainers."*

- Say whether the class is thread-safe, conditionally thread-safe, or not thread-safe, and **which lock** clients must use for compound actions (if any).
- Mark fields with `@GuardedBy`.
- The JDK is often vague (is `SimpleDateFormat` thread-safe? No. Is `java.sql.Connection`? It depends on the driver). Don't make your users guess.

---

## Cross-References

| JCIP idea | Where it was applied or measured |
|-----------|------------------------------------|
| Race shapes, atomic variables | #049 (all three races, and their fixes) |
| Visibility, `volatile`, `NoVisibility` | #048 (spin loop, litmus test) |
| Intrinsic locks, reentrancy, guarding state | #045 |
| Condition predicates, state-dependent operations | #046, #047 (Ch. 14 preview) |
| Confinement (stack) + `join` publication | #043 |
| Immutability, final fields | #003 (EJ Item 17), #048 |
| `this` escape | #025 (`-Xlint:this-escape`) |
| Documenting policy | #035 (the `SynchronizedCache` Javadoc), #050 (`HitCounter` Javadoc) |
| Delegation + invariants (`NumberRange`) | Coming: row 52 (fix an unsafe class) |
| Testing concurrent code | #050 ("test the tests"), and JCIP Ch. 12 later |

---

## Handwritten Cheat Sheet

```
CH1: benefits = cores · modelling · async · responsiveness
     hazards = SAFETY (nothing bad) · LIVENESS (something good eventually) · PERFORMANCE
     frameworks call your code on THEIR threads → your objects must be thread-safe
CH2: thread-safe = correct under any interleaving, no extra caller sync
     stateless ⇒ thread-safe · races: CHECK-THEN-ACT, READ-MODIFY-WRITE
     single var → Atomic*; related vars → ONE lock for all of them (and every access)
     reentrant intrinsic locks · narrow sync blocks · no I/O or long work under a lock
CH3: VISIBILITY (stale data, NoVisibility) · volatile = flags/publication only
     don't let `this` escape the constructor (factory method instead)
     confinement: stack · ThreadLocal · ad-hoc
     immutable (final fields, proper construction) ⇒ always thread-safe
     immutable holder + volatile reference = atomic multi-field update
     SAFE PUBLICATION: static init · volatile/AtomicRef · final field · lock · concurrent collection
CH4: design = state vars + invariants + policy
     instance confinement · Java monitor pattern (prefer a private lock)
     delegation OK only for INDEPENDENT state (NumberRange counterexample)
     adding methods: modify > compose (ImprovedList) > client-side lock (SAME lock!) > extend
     ListHelper bug (lock on helper, not list) → reproduced: 4/20 runs, rare = dangerous
     document thread-safety for clients, the sync policy for maintainers (@GuardedBy)
```

---

**Next:** row 45 — synchronized, intrinsic locks, monitor (#045)
