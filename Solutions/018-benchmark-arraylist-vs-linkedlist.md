# #018 — Solution: Benchmark ArrayList vs LinkedList Operations

**Date:** 2026-09-24  
**Problem:** [018-benchmark-arraylist-vs-linkedlist.md](../problems/018-benchmark-arraylist-vs-linkedlist.md)

> **These are real measurements**, not textbook numbers. **Machine:** Apple M4 (10 cores), macOS · **JVM:** Amazon Corretto 21.0.10+7-LTS, `-Xms2g -Xmx2g`, default GC (G1) · **JMH 1.37**, 3 × 1 s warm-up, 5 × 1 s measurement, 1 fork · **JOL 0.17** for memory. Your absolute numbers will differ; the *ratios* and *shapes* are what transfer.

---

## Part 1 — Why Not Just `System.nanoTime()`?

| # | Pitfall | How it distorts a naive loop | JMH remedy |
|---|---------|------------------------------|------------|
| 1 | **JIT warm-up** | The first thousands of calls run in the **interpreter**, then C1, then optimised C2 code. Timing from a cold start measures the compiler, not your code | `@Warmup` iterations that aren't counted |
| 2 | **Dead-code elimination** | If the result of `list.get(i)` is never used, C2 may **delete the loop entirely** and you measure ~0 ns | Return the result from the `@Benchmark` method, or `Blackhole.consume(...)`. JMH makes sure it's "used" |
| 3 | **Constant folding / hoisting** | Inputs known at compile time (a `final` local list, a constant index) let the JIT precompute or move work out of the loop | Keep inputs in a `@State` object's fields; JMH reads them as non-constant |
| 4 | **GC pauses** | One pause inside a short timing window can dominate the result | Many iterations + statistics (mean ± error); a fixed heap (`-Xms = -Xmx`); `-prof gc` to see allocation |
| 5 | **On-stack replacement (OSR)** | A long loop in `main` gets compiled mid-loop, and the OSR version is often less optimised than a normally compiled method | JMH calls the benchmark method repeatedly, so it's compiled normally |
| 6 | **One run, one JVM** | JIT decisions (inlining, profile pollution) differ between runs; one sample has no error bar | `@Fork` runs in fresh JVMs; `Cnt` and `Error` (99.9% CI) in the output |

Also: **profile pollution**. If one JVM runs the ArrayList *and* LinkedList benchmarks, call sites like `list.get` become **megamorphic** and slower for both. JMH forks per benchmark and per parameter combination by default, which is another reason to use it.

---

## Part 2 — Project Setup

### `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>list-bench</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <jmh.version>1.37</jmh.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>${jmh.version}</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>${jmh.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>0.17</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.openjdk.jmh</groupId>
                            <artifactId>jmh-generator-annprocess</artifactId>
                            <version>${jmh.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <finalName>benchmarks</finalName>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>org.openjdk.jmh.Main</mainClass>
                                </transformer>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- `jmh-generator-annprocess` is an **annotation processor**: at compile time it generates the harness classes for every `@Benchmark`.
- The **shade** plugin builds one runnable `benchmarks.jar` with `org.openjdk.jmh.Main` as the entry point. (The `ServicesResourceTransformer` and signature excludes stop merged JARs from breaking.)
- Behind a corporate artifact proxy that's unreachable, build with `mvn -s /tmp/central-settings.xml package` (see #002 Solution).

### Build & run

```bash
mvn package
java -jar target/benchmarks.jar ListBenchmark -rf json -rff results.json   # the list benchmarks
java -jar target/benchmarks.jar QueueBenchmark                             # the queue comparison
java -jar target/benchmarks.jar ListBenchmark -p size=1000 -f 1 -wi 1 -i 2 # quick smoke run
java -jar target/benchmarks.jar ListBenchmark.iterate -prof gc             # + allocation profile
java -jar target/benchmarks.jar -h                                         # all options
```

The full `ListBenchmark` run (9 benchmarks × 2 sizes × 2 impls) took about 7 minutes on the M4.

---

## Part 3 — Benchmark Code

### `ListBenchmark.java`

```java
package com.example.bench;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.List;
import java.util.ListIterator;
import java.util.SplittableRandom;
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Param;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;
import org.openjdk.jmh.infra.Blackhole;

/**
 * ArrayList vs LinkedList on the operations people argue about.
 * Every benchmark returns or consumes its result so the JIT can't delete the work (dead-code elimination).
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(value = 1, jvmArgsAppend = {"-Xms2g", "-Xmx2g"})
@State(Scope.Thread)
public class ListBenchmark {

    /** Number of elements already in the list. */
    @Param({"1000", "100000"})
    int size;

    /** Which implementation to test. */
    @Param({"ArrayList", "LinkedList"})
    String impl;

    /** How many operations each benchmark performs per invocation (keeps LinkedList runs bounded). */
    static final int OPS = 1_000;

    List<Integer> list;
    int[] randomIndexes;

    @Setup
    public void setUp() {
        list = newList();
        for (int i = 0; i < size; i++) {
            list.add(i);
        }
        SplittableRandom rnd = new SplittableRandom(42);   // fixed seed: same indexes for both impls
        randomIndexes = new int[OPS];
        for (int i = 0; i < OPS; i++) {
            randomIndexes[i] = rnd.nextInt(size);
        }
    }

    private List<Integer> newList() {
        return impl.equals("ArrayList") ? new ArrayList<>() : new LinkedList<>();
    }

    /** Build a list of `size` elements by appending. */
    @Benchmark
    public List<Integer> addLast() {
        List<Integer> l = newList();
        for (int i = 0; i < size; i++) {
            l.add(i);
        }
        return l;
    }

    /** OPS random-index reads. */
    @Benchmark
    public long getRandom() {
        long sum = 0;
        for (int idx : randomIndexes) {
            sum += list.get(idx);
        }
        return sum;
    }

    /** Full traversal with for-each (iterator). */
    @Benchmark
    public long iterate() {
        long sum = 0;
        for (int v : list) {
            sum += v;
        }
        return sum;
    }

    /** Queue-like use: OPS times add at the end + remove from the front (size stays constant). */
    @Benchmark
    public int addLastRemoveFirst() {
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            list.add(i);
            acc += list.remove(0);
        }
        return acc;
    }

    /** Insert OPS elements at the front, then remove them again (size restored). */
    @Benchmark
    public int addFirstRemoveFirst() {
        for (int i = 0; i < OPS; i++) {
            list.add(0, i);
        }
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            acc += list.remove(0);
        }
        return acc;
    }

    /**
     * Insert OPS elements in the middle THROUGH A ListIterator, then remove them the same way.
     * This is LinkedList's best case: once positioned, each insert/remove is O(1) relinking.
     * Positioning itself costs O(n) for both.
     */
    @Benchmark
    public int insertMiddleViaIterator() {
        ListIterator<Integer> it = list.listIterator(size / 2);
        for (int i = 0; i < OPS; i++) {
            it.add(i);
        }
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            acc += it.previous();
            it.remove();
        }
        return acc;
    }

    /**
     * Insert OPS elements at index size/2 BY INDEX, then remove them by index (size restored).
     * ArrayList: O(n) shift each. LinkedList: O(n) walk to the middle each (node(i)).
     */
    @Benchmark
    public int insertMiddleByIndex() {
        int mid = size / 2;
        for (int i = 0; i < OPS; i++) {
            list.add(mid, i);
        }
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            acc += list.remove(mid);
        }
        return acc;
    }

    /** Linear search for a value that is not present (worst case). */
    @Benchmark
    public boolean containsMissing() {
        return list.contains(-1);
    }

    /** Index-based loop: the classic LinkedList performance bug. Only run on the small size. */
    @Benchmark
    public void indexLoopSmallOnly(Blackhole bh) {
        if (size > 1_000) {
            return;   // on LinkedList this is O(n²): ~2.5 billion node hops at 100k
        }
        for (int i = 0; i < list.size(); i++) {
            bh.consume(list.get(i));
        }
    }
}
```

### `QueueBenchmark.java` (Part 5, Q4)

```java
package com.example.bench;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Param;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;

/** FIFO queue workload: the size stays constant while OPS elements flow through. */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(value = 1, jvmArgsAppend = {"-Xms2g", "-Xmx2g"})
@State(Scope.Thread)
public class QueueBenchmark {

    @Param({"1000", "100000"})
    int size;

    static final int OPS = 1_000;

    ArrayDeque<Integer> arrayDeque;
    LinkedList<Integer> linkedList;
    ArrayList<Integer> arrayList;

    @Setup
    public void setUp() {
        arrayDeque = new ArrayDeque<>();
        linkedList = new LinkedList<>();
        arrayList = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            arrayDeque.offerLast(i);
            linkedList.offerLast(i);
            arrayList.add(i);
        }
    }

    private static int churn(Deque<Integer> q) {
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            q.offerLast(i);
            acc += q.pollFirst();
        }
        return acc;
    }

    @Benchmark
    public int arrayDeque() {
        return churn(arrayDeque);
    }

    @Benchmark
    public int linkedList() {
        return churn(linkedList);
    }

    @Benchmark
    public int arrayListAsQueue() {
        int acc = 0;
        for (int i = 0; i < OPS; i++) {
            arrayList.add(i);
            acc += arrayList.remove(0);
        }
        return acc;
    }
}
```

Design notes:

- **`@State(Scope.Thread)` + `@Setup`**: the list is built once per trial, outside the measured region.
- **Mutating benchmarks restore the size** (add 1 000, then remove 1 000), so every invocation measures the same list size. Otherwise `size` would drift and later iterations would measure a different workload.
- **Fixed seed** (`SplittableRandom(42)`): both implementations read exactly the same indexes.
- **`OPS = 1_000` per invocation**: bounds LinkedList's slow cases. Results are **per 1 000 operations** (per invocation), except `addLast` (builds `size` elements), `iterate` and `containsMissing` (one full pass).
- **B8 only at size 1 000**: on a 100 000-element LinkedList, an index loop is O(n²), about **2.5 billion node hops** (Σ min(i, n−i) ≈ n²/4) *per invocation*, i.e. several seconds each. JMH would still finish, but it adds nothing. The size-100 000 row is the empty `return` (≈ 10⁻³ µs) and is ignored below.

---

## Part 4 — Memory Footprint (JOL)

### `MemoryFootprint.java`

```java
package com.example.bench;

import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import org.openjdk.jol.info.GraphLayout;

/** Measures retained heap of each structure holding the same 100k Integer objects (JOL). */
public class MemoryFootprint {
    public static void main(String[] args) {
        int n = 100_000;
        List<Integer> values = new ArrayList<>(n);
        for (int i = 0; i < n; i++) {
            values.add(1_000_000 + i);   // outside the Integer cache: real, distinct objects
        }

        ArrayList<Integer> arrayList = new ArrayList<>();
        LinkedList<Integer> linkedList = new LinkedList<>();
        ArrayDeque<Integer> arrayDeque = new ArrayDeque<>();
        for (Integer v : values) {
            arrayList.add(v);
            linkedList.add(v);
            arrayDeque.add(v);
        }

        // parseInstance(Object... roots) is varargs: passing the array makes each Integer a root,
        // so this measures just the 100k Integer objects (the payload shared by all three structures).
        long integers = GraphLayout.parseInstance(values.toArray()).totalSize();
        report("ArrayList", GraphLayout.parseInstance(arrayList).totalSize(), integers, n);
        report("LinkedList", GraphLayout.parseInstance(linkedList).totalSize(), integers, n);
        report("ArrayDeque", GraphLayout.parseInstance(arrayDeque).totalSize(), integers, n);
        System.out.println();
        System.out.println(GraphLayout.parseInstance(linkedList).toFootprint());
    }

    private static void report(String name, long total, long integers, int n) {
        long structure = total - integers;
        System.out.printf("%-10s total %,10d bytes | structure only %,9d bytes | %.1f bytes/element overhead%n",
                name, total, structure, (double) structure / n);
    }
}
```

```bash
java -cp target/benchmarks.jar com.example.bench.MemoryFootprint
```

### Results (100 000 distinct `Integer`s, compressed oops)

```text
ArrayList  total  2,026,880 bytes | structure only   426,880 bytes | 4.3 bytes/element overhead
LinkedList total  4,000,032 bytes | structure only 2,400,032 bytes | 24.0 bytes/element overhead
ArrayDeque total  2,034,696 bytes | structure only   434,696 bytes | 4.3 bytes/element overhead

java.util.LinkedList footprint:
     COUNT       AVG       SUM   DESCRIPTION
    100000        16   1600000   java.lang.Integer
         1        32        32   java.util.LinkedList
    100000        24   2400000   java.util.LinkedList$Node
    200001             4000032   (total)
```

### Explaining the overhead

| Structure | Layout | Arithmetic |
|-----------|--------|------------|
| `Integer` (payload, same for all three) | 12 B header + 4 B `int` value = **16 B** | 100 000 × 16 = 1 600 000 |
| `ArrayList` | one `Object[]` of capacity **106 710** (the ×1.5 growth sequence …71 140 → 106 710, #020) × **4 B** compressed references + a 16 B array header + the 24 B `ArrayList` object | 106 710 × 4 + 16 + 24 = **426 880** ✔︎ |
| `LinkedList` | one `Node` per element: 12 B header + 3 references (`item`, `next`, `prev`) × 4 B = **24 B**, + a 32 B `LinkedList` object | 100 000 × 24 + 32 = **2 400 032** ✔︎ |
| `ArrayDeque` | a circular `Object[]` with a similar amount of spare capacity | ≈ ArrayList |

The structure is **5.6× bigger** for LinkedList, and it's 100 000 separate objects for the GC to trace, versus one array.

> Measuring pitfall I hit on the first attempt: `GraphLayout.parseInstance(Object... roots)` is **varargs**, so `parseInstance(values.toArray())` treats the array's *elements* as roots and doesn't count the array itself. The first version of the program subtracted an array size that had never been added, which inflated the "structure" numbers. The lesson applies to all measurement: **check the numbers against first-principles arithmetic** (the table above) before believing them.

---

## Part 5 — Results & Analysis

### Predictions (written before running)

| # | Benchmark | ArrayList | LinkedList | Predicted winner |
|---|-----------|-----------|------------|------------------|
| B1 | addLast | amortized O(1) | O(1) + allocation | ArrayList, slightly |
| B2 | getRandom | O(1) | O(n) | ArrayList, hugely |
| B3 | iterate | O(n) | O(n) | ArrayList (cache locality) |
| B4 | add end + remove front | O(n) per `remove(0)` | O(1) | LinkedList |
| B5 | add/remove front | O(n) | O(1) | LinkedList |
| B6 | insert via iterator | O(n) shift | O(1) relink | LinkedList |
| B7 | contains (missing) | O(n) | O(n) | ArrayList (cache) |
| B8 | index loop | O(n) | O(n²) | ArrayList, hugely |
| extra | insert middle **by index** | O(n) shift | O(n) walk + O(1) link | ArrayList (a `memmove` beats pointer chasing) |

### Measured results (µs per invocation; lower is better; ± is the 99.9% confidence interval)

| Benchmark | Size | ArrayList | LinkedList | LinkedList vs ArrayList |
|-----------|-----:|----------:|-----------:|-------------------------|
| B1 addLast (build `size` elements) | 1,000 | 1.922 ± 0.068 | 2.956 ± 0.103 | 1.5× slower |
| | 100,000 | 183.6 ± 5.7 | 287.4 ± 17.7 | 1.6× slower |
| B2 getRandom (1 000 gets) | 1,000 | 0.512 ± 0.017 | 278.5 ± 6.0 | **545× slower** |
| | 100,000 | 0.611 ± 0.021 | 37,126 ± 1,805 | **60,767× slower** |
| B3 iterate (full pass) | 1,000 | 0.304 ± 0.013 | 1.511 ± 0.015 | 5.0× slower |
| | 100,000 | 30.8 ± 0.4 | 161.6 ± 0.8 | 5.2× slower |
| B7 containsMissing (full pass) | 1,000 | 0.381 ± 0.009 | 1.609 ± 0.007 | 4.2× slower |
| | 100,000 | 38.5 ± 1.5 | 171.0 ± 2.1 | 4.4× slower |
| B4 addLast + remove(0) (1 000×) | 1,000 | 59.8 ± 23.9 ⚠︎ | 5.987 ± 0.216 | **10× faster** |
| | 100,000 | 7,182 ± 94 | 6.265 ± 0.252 | **1,146× faster** |
| B5 add(0) ×1 000, remove(0) ×1 000 | 1,000 | 156.4 ± 5.7 | 6.467 ± 0.219 | **24× faster** |
| | 100,000 | 14,362 ± 259 | 6.536 ± 0.658 | **2,197× faster** |
| extra: insert/remove at `size/2` **by index** (1 000× each) | 1,000 | 106.4 ± 0.2 | 1,238 ± 16 | 11.6× slower |
| | 100,000 | 7,151 ± 24 | 137,257 ± 1,116 | 19.2× slower |
| B6 insert/remove at `size/2` **via ListIterator** (1 000× each) | 1,000 | 60.8 ± 16.1 | 7.969 ± 0.241 | **7.6× faster** |
| | 100,000 | 7,102 ± 82 | 93.0 ± 6.5 | **76× faster** |
| B8 index loop `for i… get(i)` | 1,000 | 0.078 ± 0.001 | 278.8 ± 15.2 | **3,581× slower** |

⚠︎ The error (± 23.9) on B4 ArrayList/1 000 is 40% of the score, so that one number is noisy. The 10× conclusion still stands, because the gap is far bigger than the error.

### Queue workload including `ArrayDeque` (1 000 × `offerLast` + `pollFirst`)

| Size | ArrayDeque | LinkedList | ArrayList (`add` + `remove(0)`) |
|-----:|-----------:|-----------:|--------------------------------:|
| 1,000 | **2.213 ± 0.047** | 4.811 ± 0.024 | 54.3 ± 0.9 |
| 100,000 | **2.216 ± 0.111** | 5.072 ± 0.178 | 7,130 ± 112 |

**ArrayDeque is ≈ 2.2–2.3× faster than LinkedList** at both sizes, and it stays constant with size, as an O(1) structure should. It writes into an existing array slot (no allocation) and advances an index, while LinkedList allocates a 24 B node per `offer` and leaves one for the GC per `poll`.

### Where the results disagreed with Big-O, and why

1. **B3/B7: "both O(n)", yet LinkedList is 4–5× slower.** This is **cache locality**. An ArrayList's references sit contiguously in memory, so the CPU's hardware **prefetcher** streams cache lines ahead of the loop. LinkedList must load each `Node` to learn the address of the next one (**pointer chasing**), a dependent load chain that can't be prefetched effectively. Each hop can cost a cache miss. Big-O counts *operations*; the hardware charges for *memory access patterns*.
   - (Here the nodes were allocated in order into a fresh heap, so they happen to be fairly close together. In a long-running application with GC churn they scatter, and the gap usually **grows**.)
2. **B1: "both O(1)", but LinkedList is ~1.5× slower.** Every LinkedList `add` **allocates a 24 B object** and writes 3 references. ArrayList writes one reference into an existing slot and occasionally does one big `Arrays.copyOf`.
3. **Middle insert by index: "LinkedList O(1) insert" loses 12–19×.** The O(1) link only happens after `node(i)` **walks** ~n/2 nodes, and that walk (pointer chasing) is far slower than ArrayList's `System.arraycopy` of the same number of elements (a vectorised `memmove` over contiguous memory). **Both are O(n), and the constant factors differ by more than 10×.**
4. **B6: via an iterator, LinkedList wins 7.6–76×.** Here the walk happens **once** (`listIterator(size/2)`), then each of the 2 000 operations is a true O(1) relink, while ArrayList shifts ~n/2 elements *each time*. This is LinkedList's one genuine advantage, and it only shows up if your code **holds an iterator** at the edit position.
5. **B2 at 100 000: 60 000×.** O(1) vs O(n) with n = 100 000 means each LinkedList `get` walks up to 50 000 nodes. The asymptotic difference dominates everything.
6. **ArrayList B5 at 100 000 is still fast in absolute terms:** 14 362 µs / 2 000 shifts ≈ **7 µs to shift 100 000 references** (400 KB). That's what "O(n), but a fast O(n)" means.

### Cache locality in one picture

```
ArrayList traversal:   [r0 r1 r2 r3 r4 r5 r6 r7][r8 ...]      one cache line = 16 refs (64 B, x86) or 32 refs (128 B, Apple M-series)
                        └── prefetcher loads the next line before you need it ──▶

LinkedList traversal:  Node@0x7a10 ──next──▶ Node@0x9c40 ──next──▶ Node@0x3f88 ──▶ ...
                        each hop: load node → read `next` → only then can the next load start
```

---

## Part 6 — Final Recommendation

| Use case | Choose | Evidence |
|----------|--------|----------|
| General-purpose list (append, index, iterate, search) | **`ArrayList`** | Wins B1, B2, B3, B7, B8 and middle-by-index; uses 5.6× less structure memory |
| FIFO queue / LIFO stack / sliding window | **`ArrayDeque`** | 2.2× faster than LinkedList and constant with size; no per-element allocation |
| Many inserts/removes at a position you're **already iterating at**, on large lists | `LinkedList` | B6: 76× faster at 100 000. Rare in practice; measure your own workload first |
| Priority processing | `PriorityQueue` | Different problem (heap) |

**When would I choose `LinkedList`?** Almost never. The only measured case is B6-style editing through a `ListIterator` on big lists, and even then, rebuilding into a new `ArrayList` in one pass (filter/merge) is often simpler and fast enough. For anything queue-like, `ArrayDeque` is strictly better.

---

## Key Takeaways

1. **Big-O tells you how cost *grows*; it doesn't tell you the *constant*.** Memory access patterns (contiguous vs pointer chasing) can be a 5–20× constant factor.
2. **Measure with JMH, not `nanoTime` loops**: warm-up, dead-code elimination, forks and error bars all matter.
3. **LinkedList's O(1) insertion needs you to already be at the node.** By index, you pay an O(n) walk that's slower than ArrayList's shift.
4. **`ArrayDeque` is the right queue/stack**: faster than LinkedList, no nulls, no per-element objects.
5. **Memory is performance:** 24 B per node vs ~4 B per slot means more cache misses and more GC work.
6. **Sanity-check measurements** against first-principles arithmetic (the JOL varargs mistake).
