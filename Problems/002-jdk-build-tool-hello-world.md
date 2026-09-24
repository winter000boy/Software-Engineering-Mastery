# #002 — Set Up JDK + Build Tool (Maven/Gradle), Hello World

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 2 — P1 Java Core · Java runtime & OOP foundations  
**Topics:** JDK installation, `JAVA_HOME`, `javac`/`java`, Maven lifecycle, Gradle tasks, project layout, executable JARs, `javap`

---

## Overview

Before any other P1 build task can happen you need a reliable toolchain. This task has four parts: install and verify a JDK, compile and run Hello World **by hand** (no build tool), then build the same program with **Maven** and with **Gradle**. The by-hand step matters: it shows you what the build tools automate.

---

## Part 1 — Install & Verify the JDK

**Tasks:**

1. Install an LTS JDK (21 recommended; 17 acceptable). Any vendor build is fine (Temurin, Corretto, Oracle, Zulu).
2. Set `JAVA_HOME` so it points to that JDK, and make sure `$JAVA_HOME/bin` is on your `PATH`.
3. Run and record the output of:
   ```bash
   java -version
   javac -version
   echo $JAVA_HOME
   ```
4. **Explain:** why are `java` and `javac` two separate programs? Which one belongs only to the JDK and not to a JRE?
5. If you have more than one JDK installed, list them all and explain how you switch between them.

---

## Part 2 — Hello World Without a Build Tool

Create this layout by hand:

```
hello-manual/
└── src/
    └── com/example/hello/
        └── HelloWorld.java
```

`HelloWorld` must:

- live in package `com.example.hello`
- print `Hello, World!`
- also print the running Java version (`System.getProperty("java.version")`)
- if command-line arguments are passed, greet each one instead (`Hello, Alice!`)

**Tasks:**

1. Compile into a separate `out/` directory with `javac -d`.
2. Run it with `java -cp`.
3. Package it as an **executable JAR** (with a `Main-Class` entry) and run it with `java -jar`.
4. Disassemble the class with `javap -c` and point to the bytecode instruction that calls `println`.
5. **Explain:** why does the `.class` file end up at `out/com/example/hello/HelloWorld.class` and not directly at `out/HelloWorld.class`?

---

## Part 3 — Same Program with Maven

**Tasks:**

1. Create a Maven project (`groupId` `com.example`, `artifactId` `hello-maven`) using the **standard directory layout**.
2. Configure the compiler for Java 21 using `maven.compiler.release` (not `source`/`target`) and explain why `release` is preferred.
3. Add JUnit 5 as a **test-scoped** dependency and write one test for a pure method `greeting(String name)`.
4. Make `mvn package` produce a JAR that runs with `java -jar`.
5. Run and explain what each of these does: `mvn clean`, `mvn compile`, `mvn test`, `mvn package`, `mvn verify`, `mvn install`.
6. **Explain:** what is the difference between a Maven *lifecycle*, *phase*, *plugin* and *goal*?

---

## Part 4 — Same Program with Gradle

**Tasks:**

1. Create a Gradle project with the **Kotlin DSL** (`build.gradle.kts`) and the **Gradle Wrapper**.
2. Use a **Java toolchain** to pin Java 21.
3. Apply the `application` plugin so that `./gradlew run --args="Alice Bob"` works.
4. Reuse the same JUnit 5 test.
5. **Explain:** why should you commit `gradlew` + `gradle/wrapper/*` to git, and why does the wrapper make builds reproducible?

---

## Part 5 — Compare

Fill in this table in your own words:

| Question | Maven | Gradle |
|----------|-------|--------|
| Build file format | | |
| Model (convention/declarative vs task graph) | | |
| How dependencies are declared | | |
| Incremental builds / build cache | | |
| When would you pick it? | | |

---

## Constraints

- No IDE project wizards for Part 2 — do it in a terminal so you see every step.
- Source code identical across all three variants (only build config differs).
- Every command you run should be recorded with its output (or an excerpt) in your notes.

---

## Acceptance Criteria

- [ ] `java -version` and `javac -version` report the **same** major version
- [ ] `JAVA_HOME` points to a JDK (it contains `bin/javac`)
- [ ] Manual: compiled with `javac -d out`, runs with `java -cp out ...` and with `java -jar`
- [ ] `javap -c` output inspected; `invokevirtual ... println` located
- [ ] Maven: `mvn verify` green, test passes, `java -jar target/hello-maven-1.0.0.jar Alice` prints `Hello, Alice!`
- [ ] Gradle: `./gradlew test` green, `./gradlew run --args="Alice"` works
- [ ] Comparison table filled in
- [ ] All "Explain" questions answered in your own words before reading the solution

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
