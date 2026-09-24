# #004 — Memory Areas: Heap, Stack, Metaspace & Class Loading

**Date:** 2026-08-18  
**Day:** Catch-up Day 2 of 4

---

## At a Glance (copy to handwritten notes)

- JVM memory splits into **thread-shared** (heap, metaspace) and **per-thread** (stack, PC register, native stack).
- **Heap** — all **objects** and **arrays** live here; shared by all threads; managed by **GC**.
- **Stack** — per-thread; stores **stack frames** (local vars, operand stack for bytecode); **no GC**; LIFO; fast.
- **Metaspace** (Java 8+) — **class metadata** (Class objects, methods, constant pool); in native memory; replaces PermGen.
- **PC Register** — per-thread; points to current bytecode instruction (undefined for native methods).
- **Native Method Stack** — used by JNI/native code (C/C++).
- **Class loading** = Load → Link (verify, prepare, resolve) → Initialize; done by **ClassLoader hierarchy**.
- Class loaders: **Bootstrap** (rt/core) → **Platform/Extension** → **Application/System** (classpath).
- **Parent delegation** — child asks parent first; prevents duplicate core classes.
- StackOverflowError = stack too deep (recursion); OutOfMemoryError = heap or metaspace exhausted.

---

## 1. JVM Runtime Data Areas — Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                     JVM PROCESS MEMORY                           │
│                                                                  │
│  ┌────────────────────────── SHARED ──────────────────────────┐ │
│  │  HEAP (objects, arrays)          METASPACE (class metadata)  │ │
│  │  Young Gen | Old Gen               Class*, methods, pools    │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─ THREAD 1 ─┐  ┌─ THREAD 2 ─┐  ┌─ THREAD N ─┐                 │
│  │ JVM Stack  │  │ JVM Stack  │  │ JVM Stack  │  (per thread)   │
│  │ PC Register│  │ PC Register│  │ PC Register│                  │
│  │ Native Stk │  │ Native Stk │  │ Native Stk │                  │
│  └────────────┘  └────────────┘  └────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

| Area | Shared? | Stores | GC? |
|------|---------|--------|-----|
| Heap | Yes | Objects, arrays | Yes |
| Metaspace | Yes | Class metadata | Class unloading (rare) |
| JVM Stack | Per thread | Frames, locals, refs | No (auto on pop) |
| PC Register | Per thread | Current instruction address | No |
| Native Stack | Per thread | Native (JNI) calls | No |

---

## 2. Heap — Where Objects Live

### Purpose

The heap is the **central object pool**. Every `new` creates data here (unless escape analysis + scalar replacement optimizes away — advanced topic).

```java
Person p = new Person("Alice");  // reference `p` on stack, Person object on heap
int[] arr = new int[1000];       // array object on heap
```

### Structure (generational GC model)

```
HEAP
├── Young Generation
│   ├── Eden Space        ← new objects allocated here
│   ├── Survivor S0
│   └── Survivor S1       ← objects that survive minor GC
└── Old Generation (Tenured)  ← long-lived objects promoted here
```

### Key properties

| Property | Detail |
|----------|--------|
| Shared | All threads access same heap |
| Size | `-Xms` (initial), `-Xmx` (max) |
| Allocation | Fast bump-pointer in Eden (usually) |
| Failure | `OutOfMemoryError: Java heap space` |
| Contents | Instance fields, array elements, object headers |

### What is NOT on the heap

- Local primitive variables (`int x = 5`) — on **stack**
- Static variables — in metaspace/class metadata area (values for objects still on heap)
- Method bytecode — in metaspace
- Thread stacks — separate per thread

### Object layout on heap (HotSpot, simplified)

```
┌──────────────────────────────────────┐
│ Object Header (mark word + klass ptr) │
│ Instance fields                       │
│ (padding for alignment)               │
└──────────────────────────────────────┘
```

- **Mark word** — hash code, GC age, lock state
- **Klass pointer** — points to class metadata in metaspace

---

## 3. Stack — Per-Thread Execution Memory

### Purpose

Each thread has its own **JVM stack**. Every method call pushes a **stack frame**; return pops it.

### Stack frame contains

| Component | Holds |
|-----------|-------|
| Local variable array | Primitives, object **references** (not objects themselves) |
| Operand stack | Bytecode computation workspace (push/pop values) |
| Frame data | Return address, exception table reference |

### Example trace

```java
public static void main(String[] args) {
    int a = 10;           // local `a` in main's frame
    int b = add(a, 5);    // new frame for add()
}

static int add(int x, int y) {
    return x + y;         // locals x, y in add's frame
}
```

```
Stack (bottom → top):

┌─────────────────┐
│  add(x=10,y=5)  │  ← top frame (active)
├─────────────────┤
│  main(args)     │
└─────────────────┘
```

When `add` returns, its frame is **destroyed** — locals gone automatically.

### Key properties

| Property | Detail |
|----------|--------|
| Size | `-Xss` per thread (default ~1 MB, platform-dependent) |
| GC | None — memory reclaimed when frame pops |
| Speed | Very fast allocation (just move stack pointer) |
| Failure | `StackOverflowError` (too many nested calls) |
| Thread safety | Each thread has own stack — no sharing |

### Stack vs reference

```java
String s = new String("hi");
// `s` (reference) → stack (local var in frame)
// String object    → heap
// "hi" literal     → string pool (in heap since Java 7+)
```

---

## 4. Metaspace (Java 8+) — Class Metadata

### History

| Version | Class metadata storage |
|---------|------------------------|
| Java ≤ 7 | **PermGen** (fixed size in heap, `-XX:PermSize`, `-XX:MaxPermSize`) |
| Java 8+ | **Metaspace** (native memory, grows by default) |

### What lives in metaspace

- `Class` objects (runtime representation)
- Method metadata (bytecode, exception tables)
- Constant pools (runtime structures)
- Static variables (for primitives and references; referenced objects still on heap)
- Annotations, field metadata

### Key properties

| Property | Detail |
|----------|--------|
| Location | Native memory (not Java heap) |
| Default | Unbounded (limited by OS) |
| Tuning | `-XX:MaxMetaspaceSize`, `-XX:MetaspaceSize` |
| Failure | `OutOfMemoryError: Metaspace` |
| GC | Classes unloaded when ClassLoader is collected ( uncommon for app loaders) |

### PermGen vs Metaspace — why change?

- PermGen OOM was common (too many dynamic classes — Spring, Hibernate proxies).
- Metaspace auto-grows → fewer tuning headaches.
- Better alignment with native memory management.

---

## 5. PC Register & Native Method Stack

### PC (Program Counter) Register

- One per thread.
- If thread executes Java bytecode → holds address of current instruction.
- If thread runs **native** method → PC is **undefined**.
- Smallest memory area; often overlooked in interviews.

### Native Method Stack

- Used when Java calls **native code** (JNI — Java Native Interface).
- C/C++ functions use this stack, not JVM stack.
- Example: many `System` class methods, file I/O at OS level.

---

## 6. Class Loading — Deep Dive

### Why class loading?

Bytecode on disk is useless until JVM **loads** it into memory, **links** it, and **initializes** static state.

### Three phases

```
LOADING → LINKING → INITIALIZATION
              │
              ├── Verify
              ├── Prepare
              └── Resolve (optional until first use)
```

#### Phase 1 — Loading

1. Find binary `.class` (classpath, module path, JAR).
2. Read bytes into memory.
3. Create `java.lang.Class` object in metaspace.
4. Store method/field metadata.

#### Phase 2 — Linking

| Sub-phase | What happens |
|-----------|--------------|
| **Verify** | Bytecode safety checks (stack, types, access) |
| **Prepare** | Allocate static field memory; set default values (0, null, false) |
| **Resolve** | Replace symbolic references with direct references (lazy — may defer) |

#### Phase 3 — Initialization

- Execute `<clinit>` — static block and static field assignments **in source order**.
- Thread-safe — JVM holds lock on Class object during init.
- Parent class initialized **before** child.

```java
class Parent {
    static { System.out.println("Parent static"); }
}
class Child extends Parent {
    static { System.out.println("Child static"); }
}
// Loading Child triggers: Parent init → then Child init
```

### ClassLoader hierarchy

```
                    Bootstrap ClassLoader
                    (null — loaded by JVM in native code)
                    loads: java.base modules (java.lang.*)
                              │
                    Platform/Extension ClassLoader
                    loads: extension modules
                              │
                    Application/System ClassLoader
                    loads: your classpath / module path
                              │
                    Custom ClassLoaders (optional)
                    e.g. Tomcat WebappClassLoader, OSGi
```

### Parent delegation model

When `ApplicationClassLoader.loadClass("com.app.Foo")`:

1. Ask **parent** (Platform) to load.
2. Platform asks **Bootstrap**.
3. If Bootstrap can't find it, child tries.
4. Prevents app from replacing `java.lang.String` with malicious version.

```java
ClassLoader app = ClassLoader.getSystemClassLoader();
ClassLoader platform = app.getParent();
ClassLoader bootstrap = platform.getParent(); // null — bootstrap is native
```

### When is a class loaded?

- **Lazy** — when first **actively used** (reference, `new`, static field access, static method, reflection).
- `-verbose:class` flag shows load events at startup.

### Custom class loading use cases

- Hot deployment (app servers reload WAR)
- Plugin architectures
- Bytecode generation (Spring CGLIB proxies)
- Loading from non-standard sources (network, encrypted JAR)

---

## 7. Memory Area Decision Table

| You declare… | Where it goes |
|--------------|---------------|
| `int x = 5` (local) | Stack (frame local) |
| `new Object()` | Heap (object) |
| Local ref variable | Stack (reference value) |
| Instance field | Heap (inside object) |
| `static int count` | Metaspace area for statics (value) |
| `static List list` | Metaspace holds ref; List object on heap |
| Method bytecode | Metaspace |
| `Class<?>` metadata | Metaspace |
| Thread call chain | Stack (frames) |

---

## 8. Common Errors & Tuning Flags

| Error | Cause | Flag / fix |
|-------|-------|------------|
| `StackOverflowError` | Infinite/deep recursion | Fix logic; `-Xss2m` (temporary) |
| `OutOfMemoryError: Java heap space` | Too many/live objects | `-Xmx`, fix leaks, profiling |
| `OutOfMemoryError: Metaspace` | Too many loaded classes | `-XX:MaxMetaspaceSize`, reduce dynamic proxies |
| `OutOfMemoryError: unable to create native thread` | Too many threads × stack size | Reduce threads or `-Xss` |

### Useful diagnostic commands

```bash
java -Xms512m -Xmx2g -Xss1m -XX:MaxMetaspaceSize=256m MyApp
jmap -heap <pid>          # heap summary
jstat -gc <pid> 1000      # GC stats every 1s
```

---

## 9. Pitfalls & Interview Angles

### Pitfalls

1. **"Objects are stored in variables"** — Variables hold **references**; objects on heap.
2. **"Static variables are on heap"** — Metadata/static storage in metaspace; object values they point to on heap.
3. **Confusing stack overflow with heap OOM** — Completely different causes.
4. **Forgetting each thread has its own stack** — Local vars are thread-safe by default.

### Interview favorites

| Question | Answer |
|----------|--------|
| Stack vs heap? | Stack: per-thread frames, locals, fast, no GC. Heap: shared objects, GC. |
| Where is `new int[10]`? | Array object on heap; reference on stack if local |
| PermGen vs Metaspace? | PermGen fixed in heap ≤7; Metaspace native memory ≥8 |
| Class loading phases? | Load → Link (verify, prepare, resolve) → Init |
| ClassLoader delegation? | Child delegates to parent first for security |
| What causes StackOverflowError? | Typically unbounded recursion |

---

## Handwritten Cheat Sheet

```
SHARED:  Heap (objects/arrays, GC) + Metaspace (class meta, Java 8+)
PER-THREAD: JVM Stack (frames/locals) + PC Register + Native Stack

Stack frame = locals + operand stack + return info
new → heap | local int → stack | local ref → stack pointing to heap

Heap gens: Eden → Survivor → Old (generational GC)
-Xmx heap max | -Xss stack size | -XX:MaxMetaspaceSize

Class loading:
  Load (read .class, create Class object)
  Link: Verify → Prepare → Resolve
  Init: static blocks/fields, parent first

ClassLoaders: Bootstrap → Platform → Application
  Parent delegation: ask parent first

Errors:
  StackOverflowError = deep recursion
  OOM heap = too many objects
  OOM metaspace = too many classes
```

---

**Next note:** #012 — Java Memory Model overview; stack vs heap (study)
