# #014 — Weekly Review + Spaced Repetition (Anki); Fix Gaps — Week 1

**Date:** 2026-09-24  
**Roadmap row:** 14 — P1 Java Core · Java runtime & OOP foundations · *Review*  
**Covers:** rows 1–13: #001 JVM/JRE/JDK · #002 toolchain setup · #003 Effective Java Ch. 2–4 · #004 memory areas & class loading · #005 Library domain · #006 JVM & OOP exercises · #007–#011 CLI inventory project · #012–#013 Java Memory Model

---

## At a Glance (copy to handwritten notes)

- **Review = retrieval, not re-reading.** Close the notes, answer from memory, *then* check. Anything you couldn't recall is a **gap**.
- The week in one line: **source → bytecode → class loading → memory areas → objects designed well → shared safely between threads.**
- **Self-test:** 30 questions below, answers hidden in collapsible blocks. Score yourself, and every miss gets a gap card.
- **Anki deck:** 45 cards below in pipe-separated format. Import once, review daily (≈10 min).
- **Gap-fix protocol:** for each miss, write a 3-line explanation in your own words + one code example, then make it an Anki card.
- **Readiness bar for moving on to Collections (row 17):** ≥ 80% on the self-test, and you can explain stack vs heap vs JMM without notes.

---

## 1. The Week on One Page (concept map)

```
 .java ──javac──▶ .class (bytecode) ──java──▶ JVM                                  [#001, #002]
                                              │
                    ┌─────────────────────────┼───────────────────────────┐
                    ▼                         ▼                           ▼
            CLASS LOADING              RUNTIME DATA AREAS          EXECUTION ENGINE
      Load → Link(verify,prepare,    Heap (objects, GC)           Interpreter + JIT
      resolve) → Init <clinit>       Metaspace (class metadata)   GC threads
      Bootstrap→Platform→App         Stack/thread (frames)
      parent delegation   [#004]     PC reg, native stack [#004]
                                              │
                    ┌─────────────────────────┴───────────────────────────┐
                    ▼                                                     ▼
          OBJECT DESIGN (single thread)                    SHARING OBJECTS (many threads)
   static factories, builders, DI      [#003]         JMM: visibility, ordering, atomicity [#012]
   equals/hashCode/toString/compareTo  [#003]         happens-before: lock, volatile,
   immutability, composition > inherit [#003]           start/join, transitivity
   value objects vs entities, ISP,     [#005]         thread confinement, safe publication
   strategy via interfaces     [#005, #006]           volatile = visibility, not atomicity [#013]
   layered CLI app: domain→repo→service→CLI [#007–#011]
```

---

## 2. Row-by-Row Recap

| Row | Topic | The one idea to keep | If you only re-read one section |
|----:|-------|----------------------|---------------------------------|
| 1 | JVM / JRE / JDK, bytecode | Bytecode is portable; the JVM is platform-specific. The JIT makes hot code native | #001 §3 lifecycle |
| 2 | JDK + Maven/Gradle | Packages = directories; pin the Java level (`--release`/toolchain) and the tool version (wrapper) | #002 Solution Part 2 |
| 3 | Effective Java Ch. 2–4 | Immutable + composition + `equals`/`hashCode` together | #003 Items 10, 11, 17, 18 |
| 4 | Memory areas & class loading | Heap and Metaspace are shared; stack, PC and native stack are per thread. Parent delegation | #004 §7 decision table |
| 5 | Library domain | Abstract class for an is-a family, interfaces for capabilities/strategies, value object vs entity | #005 Solution §2 |
| 6 | JVM & OOP exercises | Trace every variable: *where does it live?* | #006 Ex. 1 |
| 7–11 | CLI inventory project | Layering + interfaces at substitution points + tests make refactoring safe | #011 Solution "Design decisions" |
| 12 | JMM study | happens-before is the only guarantee of cross-thread visibility | #012 §4 |
| 13 | JMM practice | For each variable: *where?* and *who else can see it?* | #013 Part D drills |

---

## 3. Self-Test (30 questions)

**Rules:** notes closed, write answers on paper, 25 minutes, then check. 1 point each; ½ for partially right.

### A. Runtime & toolchain (rows 1–2)

1. What exactly is in a JDK that isn't in a runtime?
2. Why is Java "compiled *and* interpreted"?
3. `java -cp out HelloWorld` fails with *Could not find or load main class* but the class is in package `com.example.hello`. Why? Fix it.
4. What does `UnsupportedClassVersionError: class file version 65.0` mean?
5. Why is `--release 21` better than `-source 21 -target 21`?
6. Which bytecode instruction does `System.out.println(x)` compile to, and why that one rather than `invokestatic`?
7. Name the Maven phases in order from `compile` to `install`.

<details><summary>Answers A</summary>

1. `javac`, `jar`, `javap`, `jshell`, `jlink`, `jdeps`, `jcmd`/`jmap`/`jstack` and friends: the **development and diagnostic tools**. A runtime has only the JVM + class libraries.
2. `javac` compiles source to bytecode ahead of time. At run time the JVM **interprets** bytecode and the **JIT** compiles hot methods to native code.
3. Classes must be run by their **fully qualified name**: `java -cp out com.example.hello.HelloWorld`.
4. The class was compiled for **Java 21** (major 65) but is running on an **older** JVM. Run on ≥ 21, or recompile with `--release <older>`.
5. `--release` also compiles against **that version's API**, so using a newer-JDK method is a compile error rather than a runtime `NoSuchMethodError`.
6. `invokevirtual PrintStream.println`, because `println` is an **instance** method dispatched on the runtime type of `System.out` (which is loaded with `getstatic`).
7. `compile → test-compile → test → package → verify → install` (plus the resource/process phases in between).
</details>

### B. Memory areas & class loading (rows 4, 6)

8. Where do these live: a local `int`, a local reference variable, the object it points to, an instance field, a `static int`, method bytecode?
9. `StackOverflowError` vs `OutOfMemoryError: Java heap space`: causes and first fixes?
10. Why did Java 8 replace PermGen with Metaspace?
11. Give the three class-loading phases and the three linking sub-phases.
12. What problem does **parent delegation** prevent?
13. In what order do the static initialisers run when `new Child()` is the first use of `Child extends Parent`?
14. Why are local variables automatically thread-safe but the objects they point to not necessarily?

<details><summary>Answers B</summary>

8. Local `int` → **stack** (frame). Local reference → **stack**. The object → **heap**. Instance field → **heap** (inside the object). `static int` → class/static storage (metadata in **Metaspace**; any object a static field points to is still on the heap). Bytecode → **Metaspace**.
9. SOE = the thread's **stack** is exhausted, usually unbounded recursion: fix the recursion (only raise `-Xss` as a stop-gap). Heap OOM = too many **live** objects: find the leak with a heap dump (`jmap`, JFR) or raise `-Xmx`.
10. PermGen had a fixed maximum size inside the JVM heap and often ran out with frameworks that generate classes dynamically (proxies). Metaspace lives in **native memory** and grows by default.
11. **Load → Link → Initialise**; linking = **Verify → Prepare → Resolve**.
12. Application code replacing core classes (a fake `java.lang.String`), and the same class being loaded twice by different loaders.
13. **Parent's static initialiser first, then Child's.**
14. Each thread has its **own stack**, so a local is visible only to its thread. But the object it references lives on the **shared heap**, and if the reference escapes (a static field, a collection, another thread), other threads can reach it.
</details>

### C. Object design (rows 3, 5, 6, 7–11)

15. Give three advantages of a static factory over a constructor.
16. When do you reach for a Builder? Where do you validate?
17. What's the best way to implement a singleton, and why?
18. State the five `equals` properties. Which one breaks when `ColorPoint extends Point` adds `color` to `equals`?
19. What goes wrong if you override `equals` but not `hashCode`? Give a concrete failing call.
20. What's wrong with `return this.age - other.age;` in `compareTo`?
21. The five rules for an immutable class?
22. Why does `InstrumentedHashSet.addAll(List.of(a, b, c))` report 6 adds? What's the fix?
23. Why must a constructor never call an overridable method?
24. Abstract class vs interface: when do you choose which? (Use the Library model.)
25. Value object vs entity: define both and give an example of each from #005.
26. Why does `Library` take a `Clock` in its constructor?

<details><summary>Answers C</summary>

15. Any three of: they have **names**; they can **cache**/reuse instances; they can return **any subtype** (hidden classes like `List.of`); the returned class can **vary with the input** (`EnumSet`); the class **needn't exist yet** (service providers).
16. Many parameters, especially optional ones (≈4+), when you also want immutability and readable call sites. **Validate in the private constructor** that takes the builder, after copying the fields.
17. A **single-element `enum`**. It's safe against serialization creating a second instance and against reflection calling the private constructor.
18. Reflexive, symmetric, transitive, consistent, non-null. Adding `color` breaks **symmetry** (a naive version) or **transitivity** (the "ignore color when comparing to a plain Point" version). Fix it with composition.
19. Equal objects land in **different buckets**, so `map.put(new PhoneNumber(707,867,5309), "Jenny"); map.get(new PhoneNumber(707,867,5309))` returns **null**.
20. **Integer overflow**: for large values of opposite sign the subtraction wraps and has the wrong sign. Use `Integer.compare` or `Comparator.comparingInt`.
21. No mutators; the class can't be extended; all fields `final`; all fields `private`; exclusive access to mutable components (defensive copies).
22. `HashSet.addAll` internally calls the overridden `add`, so every element is counted twice. That's inheritance depending on implementation details. Fix: **composition** (a forwarding/wrapper class).
23. The override runs **before the subclass constructor body**, so it sees uninitialised (null/0) subclass fields.
24. **Abstract class** when subtypes share **state + behaviour** in a true is-a family (`LibraryItem` with id/title/equality). **Interface** for a **capability** or **strategy** that unrelated types can have (`Borrowable`, `MembershipPolicy`). A class can implement many interfaces but extend one class.
25. **Value object:** immutable, equal by value, no identity (`Isbn`, `Author`). **Entity:** has an identity that stays the same while its state changes, equal by id (`Member`, a `Book` copy with `ItemId`).
26. **Dependency injection of time**: tests can control "today" (a `MutableClock`), so due-date and late-fee logic is deterministic and testable.
</details>

### D. Java Memory Model (rows 12–13)

27. JMM vs JVM memory areas: what's the difference?
28. List four sources of happens-before.
29. Why is `volatile int count; count++;` still broken across threads? Give two fixes.
30. What is *unsafe publication*? Name two safe publication idioms.

<details><summary>Answers D</summary>

27. Memory **areas** (#004) describe **where** data is stored. The **JMM** (#012) is the **contract** for when a write by one thread is **visible** to another, and which reorderings are allowed.
28. Any four of: unlock → subsequent lock of the same monitor; `volatile` write → subsequent read of the same variable; `Thread.start()` → actions in the started thread; a thread's actions → another thread's successful `join()`; program order within a thread; **transitivity**; final-field freeze at the end of a constructor.
29. `count++` is **read-modify-write** (3 steps). `volatile` gives visibility, not atomicity, so two threads can read the same value and both write +1. Fix with `AtomicInteger.incrementAndGet()`, or `synchronized` around the increment (or `LongAdder` under high contention).
30. Making a reference visible to another thread **without** a happens-before edge, so the other thread may see a partially constructed object (default field values). Safe idioms: initialise from a **static initialiser**; store in a **`volatile`** field or `AtomicReference`; store in a **`final`** field of a properly constructed object; store in a field guarded by a **lock**; put it in a **thread-safe collection** (`ConcurrentHashMap`, `BlockingQueue`).
</details>

### Scoring

| Score | Meaning | Action |
|------:|---------|--------|
| 27–30 | Solid | Move on to Collections (row 17) |
| 24–26 | Good (≥ 80%) | Move on; fix the gaps this week via Anki |
| 18–23 | Shaky | Spend tomorrow's Flex slot on the weakest section before starting row 17 |
| < 18 | Not yet | Re-do #006 and #013 Part D drills before moving on |

---

## 4. Anki Deck (45 cards)

### How to import

1. Copy the block below into a file `week1.txt` (keep the first line; Anki treats `#` lines as directives).
2. Anki → **File → Import** → pick the file. It reads the `#separator:pipe` header automatically; otherwise choose **Fields separated by: Pipe**.
3. Note type **Basic (and reversed card)** is optional. Use it only for definition-style cards.
4. Review daily. Don't add more than ~20 new cards/day.

```text
#separator:pipe
#html:false
#tags:java p1 week1
JDK vs JRE vs JVM?|JVM executes bytecode; JRE = JVM + class libraries; JDK = JRE + dev tools (javac, jar, javap, jshell, jlink)
Why is Java "compiled and interpreted"?|javac compiles source to bytecode ahead of time; the JVM interprets bytecode and the JIT compiles hot methods to native code
What does the JIT do?|Compiles frequently executed (hot) bytecode into optimised native machine code at run time
Is bytecode platform-specific?|No. Bytecode is portable; the JVM implementation is platform-specific (WORA)
Class file major version for Java 8 / 11 / 17 / 21?|52 / 55 / 61 / 65
UnsupportedClassVersionError means?|Class compiled for a newer Java than the JVM running it
--release vs -source/-target?|--release also compiles against that version's API, so newer-JDK API use becomes a compile error
How must you pass the class to java -cp?|By fully qualified name, e.g. com.example.hello.HelloWorld
Why are packages directories?|The class loader maps com.example.Foo to com/example/Foo.class on the classpath or in the JAR
What does java -jar need inside the JAR?|A Main-Class entry in META-INF/MANIFEST.MF
invokestatic vs invokevirtual?|invokestatic calls a static method (no receiver); invokevirtual calls an instance method dispatched on the runtime type
How is string + compiled since Java 9?|invokedynamic makeConcatWithConstants (JEP 280), not a StringBuilder chain
Maven lifecycle vs phase vs goal?|Lifecycle = ordered phases; phase = named step; goal = plugin task bound to a phase (compiler:compile)
Why commit the Gradle/Maven wrapper?|Pins the build-tool version; no local install needed; reproducible builds
Which JVM memory areas are shared between threads?|Heap and Metaspace
Which JVM memory areas are per thread?|JVM stack, PC register, native method stack
What is in a stack frame?|Local variable array, operand stack, frame data (return address, exception table ref)
Where does new Person() live? Where does the local reference p live?|Object on the heap; reference in the current thread's stack frame
Where are static fields stored?|With class metadata (Metaspace area); objects they reference are on the heap
PermGen vs Metaspace?|PermGen: fixed-size, in heap, <= Java 7. Metaspace: native memory, grows by default, Java 8+
StackOverflowError cause?|Stack exhausted, usually unbounded or too-deep recursion
OutOfMemoryError: Java heap space cause?|Too many live objects (leak or undersized -Xmx)
Class loading phases?|Load -> Link (Verify, Prepare, Resolve) -> Initialise (<clinit>)
What happens in the Prepare phase?|Static fields get memory and default values (0, null, false)
ClassLoader hierarchy?|Bootstrap -> Platform -> Application (System) -> custom
Parent delegation?|A loader asks its parent first; prevents replacing core classes and duplicate loading
Static init order for Child extends Parent?|Parent static init runs before Child static init
Static factory advantages (3)?|Named; can cache instances; can return any subtype/hidden class
When to use a Builder?|Many (esp. optional) params and you want immutability + readable calls; validate in the private ctor
Best singleton implementation?|Single-element enum: serialization- and reflection-safe
Utility class idiom?|final class + private constructor that throws AssertionError
Can Java leak memory? Example?|Yes: obsolete references (array-backed stack not nulling popped slots), caches, listeners
Why avoid finalizers/cleaners?|Unpredictable timing, slow, security holes, finalize deprecated for removal (JEP 421)
try-with-resources: what happens to an exception thrown by close()?|It is added as a suppressed exception to the primary one
Five equals properties?|Reflexive, symmetric, transitive, consistent, x.equals(null) == false
Overriding equals without hashCode breaks what?|HashMap/HashSet: equal keys land in different buckets, so get() returns null
Why is a - b a bad comparator?|Integer overflow gives the wrong sign; use Integer.compare
Five rules of immutability?|No mutators; can't be extended; all fields final; all fields private; defensive copies of mutable parts
Why prefer composition over inheritance?|Inheritance across packages breaks encapsulation (InstrumentedHashSet counts addAll twice)
Why must constructors not call overridable methods?|The override runs before the subclass constructor, seeing uninitialised fields
Abstract class vs interface?|Abstract class: shared state + skeleton for an is-a family. Interface: capability/strategy, multiple allowed
Value object vs entity?|Value: immutable, equal by value (Isbn). Entity: stable identity, equal by id (Member)
JMM guarantees what?|Visibility, ordering and atomicity rules for shared memory between threads (the happens-before contract)
Four happens-before sources?|Unlock->lock same monitor; volatile write->read; Thread.start; thread end->join (+ program order, transitivity)
Why is volatile count++ broken?|++ is read-modify-write; volatile gives visibility, not atomicity. Use AtomicInteger or synchronized
```

---

## 5. Gap-Fix Protocol

For every question you missed or answered with < 100% confidence:

| Step | Action | Time |
|------|--------|-----:|
| 1 | Write the question on the left page of your notebook | 1 min |
| 2 | **Without looking**, write your best explanation (even if wrong) | 3 min |
| 3 | Open the source note (table in §2) and correct your explanation in a different colour | 5 min |
| 4 | Write **one tiny code example** that demonstrates it, and run it (`java File.java` single-file mode, see #002) | 5–10 min |
| 5 | Make/adjust the Anki card so the *front* is the question you missed | 1 min |
| 6 | Log it in the gap log below | 30 s |

### Gap log (fill in)

| Date | Question # | Topic | Root cause (didn't know / confused X with Y / forgot detail) | Fixed? |
|------|-----------:|-------|------------------------------------------------------------|:------:|
| | | | | ☐ |
| | | | | ☐ |
| | | | | ☐ |

### Common confusions this week (pre-emptive fixes)

| People confuse… | …with | Remember |
|-----------------|-------|----------|
| JMM | JVM memory areas | Areas = **where**. JMM = **when is it visible** |
| `volatile` | `synchronized` | volatile = visibility + ordering. synchronized = that **plus mutual exclusion** |
| "static is on the heap" | "static is in Metaspace" | The static *slot* is with class metadata; the *object it points to* is on the heap |
| Entity equality | Value equality | Would two instances with the same data be *the same thing*? If they're distinct copies, you have an entity |
| `Comparable` | `Comparator` | Comparable = the class's *natural* order (`compareTo`). Comparator = an *external* order (row 29) |
| JRE download | Runtime | No separate JRE since Java 11; build one with `jlink` |

---

## 6. Spaced Repetition Schedule

Anki schedules the cards; this is for **re-doing the exercises** (which Anki can't do):

| When | Re-do |
|------|-------|
| +1 day | Self-test §3, only the questions you missed |
| +3 days | #013 Part D (code fix drills), timed |
| +7 days | Whole self-test again. Target: 28+ |
| +14 days | Explain the §1 concept map out loud in 5 minutes (the "teach it" test) |
| +30 days | #006 exercises from scratch, without looking at the old solution |

---

## 7. Readiness Checklist for Week 2 (Collections & Generics, rows 17–41)

- [ ] Self-test ≥ 24/30
- [ ] Anki deck imported, first review done
- [ ] Can explain `equals`/`hashCode` and why `HashMap` needs both (**critical for #020**)
- [ ] Can explain Big-O of array access vs linked traversal *in principle* (warm-up for #017, #018)
- [ ] Library (#005) and CLI inventory (#007–#011) code compiles and tests pass
- [ ] Mark rows 1–14 `TRUE` in `Roadmap.csv` for the ones actually completed

---

## Handwritten Cheat Sheet

```
WEEK 1 SPINE:
source → javac → bytecode → classloader(Load/Link/Init) → JVM areas → JIT

AREAS: shared = heap + metaspace | per-thread = stack + PC + native
ERRORS: SOE = stack/recursion | OOM heap = live objects | OOM metaspace = classes

DESIGN: static factory · builder · enum singleton · DI (inject Clock!)
        equals+hashCode together · immutable by default · composition > inheritance
        abstract class = is-a family | interface = capability/strategy
        value object (by value) vs entity (by id)

JMM: happens-before = only visibility guarantee
     lock/unlock · volatile w→r · start · join · transitivity
     volatile ≠ atomic (count++ broken) → AtomicInteger / synchronized
     safe publication: static init · volatile · final · lock · concurrent collection

REVIEW LOOP: retrieve → check → gap card → re-do exercise on +1/+3/+7/+14/+30
```

---

**Next note:** #015 — Research paper: *On Designing and Deploying Internet-Scale Services* (J. Hamilton)
