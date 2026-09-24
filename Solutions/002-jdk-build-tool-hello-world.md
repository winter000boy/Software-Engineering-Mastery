# #002 — Solution: Set Up JDK + Build Tool (Maven/Gradle), Hello World

**Date:** 2026-09-24  
**Problem:** [002-jdk-build-tool-hello-world.md](../problems/002-jdk-build-tool-hello-world.md)

> Every command and output below was run and verified on macOS (Apple Silicon) with Amazon Corretto 21.0.10, Maven 3.9.11 and Gradle 8.14.3.

---

## Part 1 — Install & Verify the JDK

### Install (macOS)

Pick **one** method:

```bash
# Option A — Homebrew (Temurin)
brew install --cask temurin@21

# Option B — SDKMAN (easiest way to juggle several JDKs)
curl -s "https://get.sdkman.io" -o sdkman-install.sh   # read it first, then:
bash sdkman-install.sh
sdk install java 21.0.10-amzn
sdk use java 21.0.10-amzn

# Option C — IntelliJ: File → Project Structure → SDKs → Download JDK
```

Linux: `sudo apt install openjdk-21-jdk` (Debian/Ubuntu) or SDKMAN. Windows: MSI installer from Adoptium/Corretto and tick "Set JAVA_HOME".

### Set `JAVA_HOME` (zsh on macOS)

```bash
# ~/.zshrc
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
export PATH="$JAVA_HOME/bin:$PATH"
```

`/usr/libexec/java_home` is a macOS helper that finds installed JDKs. Reload the shell with `source ~/.zshrc`.

### Verify

```text
$ java -version
openjdk version "21.0.10" 2026-01-20 LTS
OpenJDK Runtime Environment Corretto-21.0.10.7.1 (build 21.0.10+7-LTS)
OpenJDK 64-Bit Server VM Corretto-21.0.10.7.1 (build 21.0.10+7-LTS, mixed mode, sharing)

$ javac -version
javac 21.0.10
```

### Q: Why are `java` and `javac` separate programs?

| Tool | Job | Ships in |
|------|-----|----------|
| `javac` | **Compiler**: turns `.java` source into `.class` bytecode | JDK only |
| `java` | **Launcher**: starts a JVM, loads classes and runs `main` | JDK and JRE |

They run at different times (build time vs. run time) and a production server only needs the runtime. `javac`, `jar`, `javap`, `jshell` and `jlink` are the **development** tools that make a JDK a JDK. (Since Java 11 there's no separate JRE download; you build a slim runtime with `jlink` if you need one.)

### Q: Several JDKs — list them and switch between them

```text
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (3):
    21.0.10 (arm64) "Amazon.com Inc." - "Amazon Corretto 21"  .../corretto-21.0.10/Contents/Home
    17.0.18 (arm64) "Amazon.com Inc." - "Amazon Corretto 17"  .../corretto-17.0.18/Contents/Home
    17.0.12 (arm64) "Oracle Corporation" - "Java SE 17.0.12"  /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home
```

Switching:

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)   # this shell only
sdk use java 17.0.18-amzn                          # SDKMAN, this shell only
sdk default java 21.0.10-amzn                      # SDKMAN, permanently
```

### ⚠️ A real gotcha found while verifying this solution

On the verification machine, `java` on the PATH was **21** but Maven reported:

```text
$ mvn -v
Apache Maven 3.9.11
Java version: 25.0.2, vendor: Homebrew, runtime: /opt/homebrew/Cellar/openjdk/25.0.2/...
```

Homebrew's Maven depends on Homebrew's `openjdk` formula and uses it when `JAVA_HOME` isn't exported. So **the JDK that Maven builds with can differ from the one you run with.** To avoid it:

- Always `export JAVA_HOME` in your shell profile (`mvn` honours it).
- Pin the language level in the build (`maven.compiler.release` / Gradle toolchains). Then a newer JDK still produces Java 21 bytecode and rejects APIs newer than 21.
- Check with `mvn -v` and `./gradlew -v`, not just `java -version`.

---

## Part 2 — Hello World Without a Build Tool

### `src/com/example/hello/HelloWorld.java`

```java
package com.example.hello;

public class HelloWorld {

    static String greeting(String name) {
        if (name == null || name.isBlank()) {
            return "Hello, World!";
        }
        return "Hello, " + name.strip() + "!";
    }

    public static void main(String[] args) {
        if (args.length == 0) {
            System.out.println(greeting(null));
        } else {
            for (String name : args) {
                System.out.println(greeting(name));
            }
        }
        System.out.println("Running on Java " + System.getProperty("java.version"));
    }
}
```

`greeting` is a separate, pure method so that it can be unit-tested without capturing `System.out`.

### Compile & run

```text
$ javac -d out src/com/example/hello/HelloWorld.java
$ find out -type f
out/com/example/hello/HelloWorld.class

$ java -cp out com.example.hello.HelloWorld
Hello, World!
Running on Java 21.0.10

$ java -cp out com.example.hello.HelloWorld Alice Bob
Hello, Alice!
Hello, Bob!
Running on Java 21.0.10
```

- `-d out` writes class files to `out/` instead of next to the source.
- `-cp out` (classpath) tells the JVM where to look for classes. Pass the **fully qualified** name (`com.example.hello.HelloWorld`), not a file path.
- Shortcut (Java 11+): `java src/com/example/hello/HelloWorld.java` compiles in memory and runs a single-file program without writing any `.class` file.

### Executable JAR

```text
$ jar --create --file hello.jar --main-class com.example.hello.HelloWorld -C out .
$ java -jar hello.jar Alice
Hello, Alice!
Running on Java 21.0.10

$ unzip -p hello.jar META-INF/MANIFEST.MF
Manifest-Version: 1.0
Created-By: 21.0.10 (Amazon.com Inc.)
Main-Class: com.example.hello.HelloWorld
```

A JAR is just a ZIP file with a `META-INF/MANIFEST.MF` inside. `java -jar` reads the `Main-Class` entry to know where to start. `-C out .` means "change into `out/` and add everything there", so package folders stay at the root of the JAR.

### `javap -c` — find the `println`

```text
$ javap -c -cp out com.example.hello.HelloWorld
  public static void main(java.lang.String[]);
    Code:
       0: aload_0
       1: arraylength
       2: ifne          18
       5: getstatic     #23   // Field java/lang/System.out:Ljava/io/PrintStream;
       8: aconst_null
       9: invokestatic  #29   // Method greeting:(Ljava/lang/String;)Ljava/lang/String;
      12: invokevirtual #34   // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      ...
      52: getstatic     #23   // Field java/lang/System.out:Ljava/io/PrintStream;
      55: ldc           #40   // String java.version
      57: invokestatic  #42   // Method java/lang/System.getProperty:(Ljava/lang/String;)Ljava/lang/String;
      60: invokedynamic #45,0 // InvokeDynamic #1:makeConcatWithConstants:(Ljava/lang/String;)Ljava/lang/String;
      65: invokevirtual #34   // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      68: return
```

How to read it:

| Instruction | Meaning |
|-------------|---------|
| `getstatic System.out` | Push the static `PrintStream` onto the operand stack |
| `invokestatic greeting` | Call a **static** method; no receiver object |
| `invokevirtual println` | Call an **instance** method, dispatched on the runtime type of the receiver (polymorphism) |
| `invokedynamic makeConcatWithConstants` | `"Running on Java " + x`. Since Java 9, string `+` compiles to an indy call (JEP 280) instead of a `StringBuilder` chain, so the JVM can choose the best concatenation strategy at run time |
| `ifne` / `if_icmpge` / `goto` | The `if` and the enhanced-for loop, lowered to jumps |

This connects back to note #001 (bytecode) and #004 (operand stack inside a stack frame).

### Q: Why `out/com/example/hello/HelloWorld.class`?

A class's **fully qualified name** is `com.example.hello.HelloWorld`. The class loader maps package segments to directories (and to paths inside a JAR). When asked for `com.example.hello.HelloWorld` it looks for `<classpath-root>/com/example/hello/HelloWorld.class`. If the file were at `out/HelloWorld.class`, loading would fail with `NoClassDefFoundError` (wrong name). `javac -d` builds the package directories for you.

---

## Part 3 — Maven

### Layout (standard directory layout)

```
hello-maven/
├── pom.xml
└── src/
    ├── main/java/com/example/hello/HelloWorld.java
    └── test/java/com/example/hello/HelloWorldTest.java
```

Generate it with `mvn archetype:generate -DgroupId=com.example -DartifactId=hello-maven -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false`, or just create the folders by hand.

### `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>hello-maven</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.11.4</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.4.2</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.example.hello.HelloWorld</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

Plugin versions are pinned on purpose. Without that, Maven uses whatever defaults its super-POM carries. Those are old (for example an old Surefire that doesn't run JUnit 5 cleanly) and they change between Maven versions.

### `HelloWorldTest.java`

```java
package com.example.hello;

import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

class HelloWorldTest {

    @Test
    void greetsByName() {
        assertEquals("Hello, Alice!", HelloWorld.greeting("Alice"));
    }

    @Test
    void trimsWhitespace() {
        assertEquals("Hello, Bob!", HelloWorld.greeting("  Bob  "));
    }

    @Test
    void fallsBackToWorldForNullOrBlank() {
        assertEquals("Hello, World!", HelloWorld.greeting(null));
        assertEquals("Hello, World!", HelloWorld.greeting("   "));
    }
}
```

The test sits in the **same package** as `HelloWorld`, so it can call the package-private `greeting` method without making it `public`.

### Build & run

```text
$ mvn verify
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO] Building jar: .../hello-maven/target/hello-maven-1.0.0.jar
[INFO] BUILD SUCCESS

$ java -jar target/hello-maven-1.0.0.jar Alice
Hello, Alice!
Running on Java 21.0.10
```

### Q: Why `release` instead of `source`/`target`?

`-source 21 -target 21` only controls the language level and the bytecode version. It still compiles against **the JDK you're running**. So on JDK 25 you could call an API added in 22+ and get a JAR that crashes on a Java 21 runtime with `NoSuchMethodError`. `--release 21` (Java 9+) also compiles against the **Java 21 API signatures**, so that mistake becomes a compile error. It's one flag that does the right thing.

### Common commands

| Command | What it does |
|---------|--------------|
| `mvn clean` | Deletes `target/` (the separate *clean* lifecycle) |
| `mvn compile` | Compiles `src/main/java` → `target/classes` |
| `mvn test` | compile → test-compile → run unit tests (Surefire) |
| `mvn package` | …test → build the JAR in `target/` |
| `mvn verify` | …package → integration tests & checks (Failsafe, quality plugins) |
| `mvn install` | …verify → copy the JAR into `~/.m2/repository` so other local projects can depend on it |
| `mvn deploy` | …install → upload to a remote repository (Nexus/Artifactory/CodeArtifact) |

Running a phase runs **every phase before it** in the same lifecycle, so `mvn package` always runs the tests. Skipping them takes an explicit `-DskipTests`, and that should be rare.

### Q: Lifecycle vs phase vs plugin vs goal

| Concept | Definition | Example |
|---------|------------|---------|
| **Lifecycle** | An ordered sequence of phases | `default`, `clean`, `site` |
| **Phase** | A named step in a lifecycle; it does nothing by itself | `compile`, `test`, `package` |
| **Plugin** | A JAR of build logic | `maven-compiler-plugin` |
| **Goal** | One task inside a plugin, **bound** to a phase | `compiler:compile` bound to `compile`; `surefire:test` bound to `test` |

"Convention over configuration": `packaging=jar` gives you sensible default goal bindings, so a nearly empty POM can already build a JAR.

### Dependency scopes (quick reference)

| Scope | On compile classpath? | On test classpath? | Packaged / at runtime? |
|-------|:---:|:---:|:---:|
| `compile` (default) | ✅ | ✅ | ✅ |
| `provided` (e.g. Servlet API) | ✅ | ✅ | ❌ (the container supplies it) |
| `runtime` (e.g. a JDBC driver) | ❌ | ✅ | ✅ |
| `test` (e.g. JUnit) | ❌ | ✅ | ❌ |

### Corporate networks: when downloads fail

When verifying this solution, the machine's `~/.m2/settings.xml` mirrored everything through a company artifact proxy that couldn't be reached from the current network:

```text
[ERROR] Plugin org.apache.maven.plugins:maven-surefire-plugin:3.5.2 ... could not be resolved:
Could not transfer artifact ... from/to <company-proxy> ... Connect timed out
```

Fix it without editing your real settings: connect to the VPN, **or** run once with a throwaway settings file:

```bash
printf '<settings/>\n' > /tmp/central-settings.xml
mvn -s /tmp/central-settings.xml verify       # uses Maven Central directly
```

---

## Part 4 — Gradle (Kotlin DSL + Wrapper)

### Layout

```
hello-gradle/
├── settings.gradle.kts
├── build.gradle.kts
├── gradlew  gradlew.bat
├── gradle/wrapper/gradle-wrapper.jar
├── gradle/wrapper/gradle-wrapper.properties
└── src/  (same main/test layout as Maven)
```

### `settings.gradle.kts`

```kotlin
rootProject.name = "hello-gradle"
```

### `build.gradle.kts`

```kotlin
plugins {
    application
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:5.11.4"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

application {
    mainClass = "com.example.hello.HelloWorld"
}

tasks.test {
    useJUnitPlatform()
    testLogging {
        events("passed", "failed", "skipped")
    }
}

tasks.jar {
    manifest {
        attributes["Main-Class"] = application.mainClass
    }
}
```

Line by line:

- `application` also applies `java` and adds the `run`, `installDist` and `distZip` tasks.
- **Toolchain**: Gradle finds (or downloads) a JDK 21 for compiling and testing, **whatever JDK runs Gradle itself**. This is the Gradle answer to the Maven/`JAVA_HOME` mismatch above.
- `platform("org.junit:junit-bom:...")` imports a **BOM** (bill of materials) that fixes the versions of all JUnit modules, so the individual lines need no version.
- `useJUnitPlatform()` is required. Without it, Gradle looks for JUnit 4 tests and silently runs **zero** tests.

### Build & run

```text
$ gradle wrapper --gradle-version 8.14.3     # one time, needs a local Gradle install
$ ./gradlew test                              # 3 tests, 0 failures (build/test-results/test/*.xml)
$ ./gradlew run --args="Alice Bob"
Hello, Alice!
Hello, Bob!
Running on Java 21.0.10

$ ./gradlew jar
$ java -jar build/libs/hello-gradle-1.0.0.jar Carol
Hello, Carol!
Running on Java 21.0.10
```

### Q: Why commit the wrapper?

`gradle/wrapper/gradle-wrapper.properties` pins the exact Gradle version (`distributionUrl=.../gradle-8.14.3-bin.zip`). `./gradlew` downloads that version on first use and caches it. So:

- teammates and CI need **no** Gradle installation, only a JDK
- everyone builds with the **same** Gradle version, so there are no "works on my machine" plugin or DSL differences
- upgrading Gradle is a one-line reviewed change: `./gradlew wrapper --gradle-version X`

Maven has the same idea in the **Maven Wrapper** (`mvn wrapper:wrapper` → `./mvnw`). Use it in real projects.

### Gradle configurations vs Maven scopes

| Gradle | ≈ Maven | Notes |
|--------|---------|-------|
| `implementation` | `compile` | Not leaked to consumers' compile classpath (faster, cleaner builds) |
| `api` (needs `java-library`) | `compile` | Leaked to consumers, for types in your public API |
| `compileOnly` | `provided` | |
| `runtimeOnly` | `runtime` | |
| `testImplementation` | `test` | |

---

## Part 5 — Comparison

| Question | Maven | Gradle |
|----------|-------|--------|
| Build file format | XML (`pom.xml`) | Kotlin or Groovy DSL (code) |
| Model | Fixed lifecycle of phases; declarative, strong conventions | Directed acyclic **graph of tasks**; conventions via plugins, fully programmable |
| Dependencies | `<dependency>` + `<scope>` | `implementation(...)` / `testImplementation(...)` configurations |
| Incremental / cache | Limited; rebuilds modules (extensions exist) | Up-to-date checks, local + remote **build cache**, configuration cache, daemon |
| Speed on big builds | Slower | Usually much faster |
| Learning curve | Low, predictable | Higher, since arbitrary build logic is possible |
| Pick it when… | You want boring, standard, easy-to-read builds (most Spring Boot services) | Big multi-module builds, Android, custom build logic, build speed matters |

For this roadmap, **Maven** is used for the P1–P3 projects (simplest, and what most Java backend job postings expect), with Gradle as a known alternative.

---

## Troubleshooting Cheat Table

| Symptom | Cause | Fix |
|---------|-------|-----|
| `javac: command not found` | Only a JRE is installed, or `PATH` is missing `$JAVA_HOME/bin` | Install a JDK; fix `PATH` |
| `Error: Could not find or load main class HelloWorld` | Used the file path or a short name; wrong `-cp` | `java -cp out com.example.hello.HelloWorld` |
| `UnsupportedClassVersionError ... class file version 65.0` | Compiled with a newer JDK (65 = Java 21) than the runtime | Run on ≥ that JDK, or compile with `--release <older>` |
| `no main manifest attribute, in x.jar` | JAR built without `Main-Class` | `--main-class` / jar-plugin `<mainClass>` / `tasks.jar.manifest` |
| Gradle: "0 tests" / tests silently skipped | Missing `useJUnitPlatform()` | Add it to `tasks.test` |
| Maven uses a different Java than `java -version` | `JAVA_HOME` not exported (Homebrew Maven uses its own `openjdk`) | `export JAVA_HOME=...`; check `mvn -v` |
| Maven `Could not transfer artifact ... Connect timed out` | Mirror/proxy in `settings.xml` unreachable | VPN, or `mvn -s /tmp/central-settings.xml` |

Class-file major versions to remember: **52 = Java 8, 55 = 11, 61 = 17, 65 = 21.**

---

## Key Takeaways

1. **JDK = JRE + dev tools** (`javac`, `jar`, `javap`, `jshell`, `jlink`). Production needs a runtime; development needs the JDK.
2. **Packages are directories.** The fully qualified name decides where the `.class` must live and what you pass to `java`.
3. **Build tools automate** compile → test → package → publish and fetch dependencies. Knowing the manual steps makes build errors readable.
4. **Pin everything:** `--release`/toolchains for the Java level, plugin versions, and the Gradle/Maven wrapper for the tool version.
5. **The JDK that builds ≠ always the JDK that runs.** Check `mvn -v` / `./gradlew -v`.
6. `javap -c` makes bytecode concrete: `invokestatic` vs `invokevirtual` is static vs. dynamic dispatch, and string `+` has been `invokedynamic` since Java 9.
