# #001 — JVM / JRE / JDK, Bytecode & How Java Runs

**Date:** 2026-08-18  
**Day:** Catch-up Day 1 of 4

---

## At a Glance (copy to handwritten notes)

- **JDK** = full dev kit (compiler + tools + JRE). **JRE** = runtime only (JVM + libraries). **JVM** = executes bytecode.
- Java is **Write Once, Run Anywhere (WORA)** — source → `.class` bytecode → JVM on any OS interprets/JIT-compiles it.
- **`.java`** → `javac` compiles → **`.class` bytecode** → JVM loads → verify → execute (interpreter + JIT).
- Bytecode is **platform-independent**; JVM is **platform-specific** (Windows JVM, Linux JVM, etc.).
- JVM does: **class loading**, **bytecode verification**, **memory management (GC)**, **execution engine** (interpreter + JIT + GC threads).
- **JIT (Just-In-Time)** compiles hot bytecode to native machine code for speed; **interpreter** runs cold code immediately.
- Java is **compiled + interpreted** — not purely one or the other.
- Entry point: `public static void main(String[] args)` — JVM calls this after loading the class.

---

## 1. The Three Layers: JDK, JRE, JVM

```
┌─────────────────────────────────────────────┐
│                    JDK                       │
│  ┌─────────────────────────────────────┐    │
│  │              JRE                     │    │
│  │  ┌─────────────────────────────┐    │    │
│  │  │            JVM               │    │    │
│  │  └─────────────────────────────┘    │    │
│  │  + Core libraries (rt.jar → modules)│    │
│  │  + java, javaw launchers            │    │
│  └─────────────────────────────────────┘    │
│  + javac (compiler)                          │
│  + jar, javadoc, jdb, jconsole, jlink…       │
└─────────────────────────────────────────────┘
```

### JVM — Java Virtual Machine

| Aspect | Detail |
|--------|--------|
| **What** | Abstract machine spec + native implementation per OS/CPU |
| **Job** | Load `.class` files, verify bytecode, allocate memory, run code, run GC |
| **Analogy** | CPU emulator tailored for Java bytecode instruction set |
| **Spec** | Defined by Oracle/OpenJDK — any vendor can implement (HotSpot, GraalVM, OpenJ9) |

### JRE — Java Runtime Environment

| Aspect | Detail |
|--------|--------|
| **What** | JVM + standard class libraries + supporting files |
| **Job** | Everything needed to **run** Java apps — no compiler |
| **Status** | **Removed from Java 11+** as separate download; folded into JDK |
| **Libraries** | `java.lang`, `java.util`, I/O, networking, collections, etc. |

### JDK — Java Development Kit

| Aspect | Detail |
|--------|--------|
| **What** | JRE + development tools |
| **Job** | Write, compile, debug, document, package Java code |
| **Key tools** | `javac`, `java`, `jar`, `javadoc`, `jdb`, `jshell` (9+), `jlink` (9+) |
| **Rule** | You need JDK to **develop**; end users historically needed only JRE |

> **Interview one-liner:** JDK ⊃ JRE ⊃ JVM. Since Java 11, ship the JDK; JRE is no longer a separate product.

---

## 2. What Is Bytecode?

### Definition

**Bytecode** = platform-neutral intermediate representation stored in `.class` files.  
It is **not** source code and **not** native machine code — it is JVM instruction set opcodes.

### Why bytecode exists

1. **Portability** — same `.class` runs on Windows, Linux, macOS with the right JVM.
2. **Security** — JVM verifies bytecode before execution (no arbitrary memory writes).
3. **Optimization** — JVM can profile and JIT-compile hot paths to native code.
4. **Late binding** — classes loaded at runtime enable dynamic behavior.

### `.class` file structure (high level)

| Section | Purpose |
|---------|---------|
| Magic number | `0xCAFEBABE` — identifies valid class file |
| Version | Major/minor — must match JVM version |
| Constant pool | Literals, class/method/field name references |
| Access flags | public, final, interface, etc. |
| This class / super class | Inheritance info |
| Interfaces | Implemented interfaces |
| Fields | Field descriptors |
| Methods | Bytecode instructions + exception table |
| Attributes | Source file name, line numbers, annotations |

### Viewing bytecode

```bash
javac HelloWorld.java          # produces HelloWorld.class
javap -c -v HelloWorld         # disassemble — see opcodes
```

Example snippet from `javap -c`:

```
0: aload_0
1: invokespecial #1   // Method java/lang/Object."<init>":()V
4: getstatic     #2   // Field java/lang/System.out:Ljava/io/PrintStream;
7: ldc           #3   // String Hello
9: invokevirtual #4   // Method java/io/PrintStream.println:(Ljava/lang/String;)V
12: return
```

Each line is a **bytecode instruction** (opcode + operands).

---

## 3. How Java Runs — End-to-End Lifecycle

### Step-by-step flow

```
┌──────────┐    javac     ┌──────────┐    java      ┌──────────┐
│ .java    │ ──────────►  │ .class   │ ──────────►  │   JVM    │
│ (source) │  compile     │(bytecode)│   launch     │ (execute)│
└──────────┘              └──────────┘              └──────────┘
                                                           │
                     ┌─────────────────────────────────────┤
                     ▼                                     ▼
              Class Loader                          Execution Engine
              (load .class)                    (Interpreter + JIT + GC)
                     │
                     ▼
              Bytecode Verifier
              (safety checks)
                     │
                     ▼
              Runtime Data Areas
              (heap, stacks, metaspace…)
```

### Phase 1 — Write & Compile

```java
// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}
```

```bash
javac HelloWorld.java   # compiler (part of JDK) → HelloWorld.class
```

- `javac` does **lexical analysis → parsing → semantic analysis → bytecode generation**.
- One public class per file; filename must match public class name.
- Inner classes produce separate `Outer$Inner.class` files.

### Phase 2 — Launch JVM

```bash
java HelloWorld    # no .class extension
```

What happens internally:

1. **`java` launcher** (native stub) starts a JVM process.
2. JVM bootstraps — allocates memory, starts internal threads.
3. **Bootstrap class loader** loads core classes (`java.lang.Object`, etc.).
4. **Application class loader** loads `HelloWorld` from classpath/module path.
5. JVM finds `main` method signature: `public static void main(String[] args)`.
6. Execution begins.

### Phase 3 — Class Loading (summary — deep dive in #004)

Three phases per class: **Loading → Linking → Initialization**

- **Loading** — read binary, create `Class` object in metaspace.
- **Linking** — verify bytecode, prepare static fields, resolve symbolic references.
- **Initialization** — run static initializers (`<clinit>`).

### Phase 4 — Bytecode Verification

Verifier checks (security + correctness):

- No stack overflow/underflow
- Type safety on operand stack
- No illegal field/method access
- Valid branch targets

> This is why Java is safer than running raw native code from untrusted sources (with SecurityManager / module restrictions in enterprise setups).

### Phase 5 — Execution Engine

Two modes work **together**:

| Component | Role | When used |
|-----------|------|-----------|
| **Interpreter** | Executes bytecode line-by-line | Startup, cold code, rarely called methods |
| **JIT Compiler** (C1/C2 in HotSpot) | Compiles hot methods to native machine code | Methods called many times ("hot spots") |
| **GC threads** | Reclaim unreachable heap objects | Concurrent / stop-the-world depending on collector |

**HotSpot JVM** name comes from profiling which methods are "hot" and compiling them.

### Phase 6 — Program Termination

- All non-daemon threads finish → JVM exits.
- `System.exit(n)` forces immediate exit.
- Uncaught exception in main → exit with error.

---

## 4. WORA — Write Once, Run Anywhere

| Layer | Platform dependent? |
|-------|---------------------|
| `.java` source | No (same everywhere) |
| `.class` bytecode | No (same everywhere) |
| JVM implementation | **Yes** (OS + CPU specific) |
| Native libraries (JNI) | **Yes** |

**Key insight:** You ship bytecode; each platform's JVM translates it to that machine's native instructions.

---

## 5. JVM Implementations (know for interviews)

| JVM | Vendor | Notes |
|-----|--------|-------|
| **HotSpot** | Oracle/OpenJDK | Default for most Java apps; C1 + C2 tiers |
| **GraalVM** | Oracle | Polyglot; Graal JIT; native-image (AOT) |
| **OpenJ9** | Eclipse/IBM | Lower memory footprint; common in containers |
| **Azul Zing / Prime** | Azul | Low-latency GC (C4, Shenandoah-like) |

All must conform to the **JVM Specification** (JSR) to run valid bytecode.

---

## 6. Compilation Models Compared

| Language model | Compile time | Run time |
|----------------|-------------|----------|
| **C/C++** | Full AOT to native | Direct CPU execution |
| **Python** | Interpreted (bytecode in CPython) | Interpreter loop |
| **Java** | Compile to bytecode | Interpreter + JIT to native |
| **Java (Graal native-image)** | AOT to native binary | No JVM at runtime |

Java chose bytecode + JVM for **balance of portability, safety, and performance**.

---

## 7. Key Tools Quick Reference

| Command | Purpose |
|---------|---------|
| `javac File.java` | Compile source → bytecode |
| `java ClassName` | Run main class |
| `java -jar app.jar` | Run packaged app |
| `javap -c ClassName` | Disassemble bytecode |
| `java -version` | JVM version info |
| `jar cvf app.jar *.class` | Create JAR archive |
| `jps` | List running Java processes |
| `jcmd <pid> help` | Diagnostic commands |

---

## 8. Common Pitfalls & Interview Angles

### Pitfalls

1. **"Java is interpreted only"** — Wrong. JIT makes hot code as fast as native in many cases.
2. **Confusing JDK vs JRE** — Since Java 11, just say JDK for both dev and runtime.
3. **Thinking `.class` is encrypted** — It's easily decompiled (`javap`, JD-GUI); don't treat it as secret.
4. **Wrong Java on PATH** — `java` vs `javac` version mismatch causes confusing errors.

### Interview questions

| Question | Answer sketch |
|----------|---------------|
| Difference JDK, JRE, JVM? | Nested containment; JDK has compiler + tools |
| Is Java compiled or interpreted? | Both — javac compiles; JVM interprets + JIT |
| What is bytecode? | JVM opcodes in `.class`; platform-independent |
| Why WORA works? | Bytecode is universal; JVM is per-platform |
| What does JVM do besides run code? | Memory mgmt, GC, verification, class loading |
| What is JIT? | Compiles frequently executed bytecode to native code at runtime |

---

## Handwritten Cheat Sheet

```
JDK = dev tools + JRE
JRE = JVM + libs (gone as separate product since Java 11)
JVM = runs bytecode, GC, class loading, verification

Flow: .java --javac--> .class --JVM--> execute
Bytecode = CAFEBABE .class opcodes (not source, not native)

JVM internals:
  ClassLoader → Verifier → Runtime memory → Execution Engine
  Execution = Interpreter (cold) + JIT (hot) + GC

WORA: same .class, different JVM per OS/CPU
HotSpot = default JVM; profiles "hot" methods → native code

main() = entry point; JVM calls it after class init
javac = compile | java = run | javap = disassemble
```

---

**Next note:** #003 — Effective Java reading notes (Ch. 2–4)
