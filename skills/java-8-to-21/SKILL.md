---
name: java-8-to-21
description: Upgrade a Java 8 codebase to Java 21 (LTS) — module path, records, pattern matching, sealed classes, virtual threads, switch expressions, the dependency upgrades (Spring Boot 2 → 3) that ride along, and the JEPs you'll actually use day-to-day. Use when on Java 8 and ready for the multi-year-overdue jump, or staged migration through 11/17.
---

# Upgrade Java 8 → 21

Java 8 LTS support ended for many vendors; Java 11, 17, 21 are the LTS waypoints. The 8 → 21 jump unlocks records, virtual threads, pattern matching — language features that meaningfully improve code. The trade-off: dependency upgrades (Spring Boot 2 → 3, Hibernate 5 → 6) ride along.

## When to use

- Codebase still on Java 8 (yes, this is common in 2026).
- Considering 8 → 11 or 11 → 17 — same pattern; less benefit.
- Greenfield modules that should be 21 from the start.

## When NOT to use

- Stuck on a vendored Java 8 by customer requirement (regulated, embedded).
- Using libraries that explicitly don't support 17+ (rare; usually a fork exists).
- Java 8 with Spring Boot 1.x — multiple sequential upgrades (Spring Boot 1 → 2 → 3 + Java 8 → 21). Stage carefully.

## The waypoints

| Java | Released | LTS until | Highlights |
|---|---|---|---|
| 8 | 2014 | 2030 (paid) | Lambdas, Streams |
| 11 | 2018 | 2026 | `var`, `HttpClient`, modules |
| 17 | 2021 | 2029 | Records, sealed, pattern matching, text blocks |
| 21 | 2023 | 2031 | Virtual threads, pattern matching for switch |

Recommended path: **8 → 17 → 21**, in two PRs ideally. Skipping 11 is fine; 11 was a tooling milestone but few language features.

## What you'll actually use

### Records (Java 16+, available in 17)

```java
// Before: 30 lines of boilerplate
public class Money {
    private final long amountPaise;
    private final String currency;
    public Money(long amountPaise, String currency) { ... }
    public long getAmountPaise() { return amountPaise; }
    public String getCurrency() { return currency; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// After: 1 line
public record Money(long amountPaise, String currency) {}
```

DTOs, value objects, immutable data — all become one-liners. Aggressive migration here pays off fast.

### Pattern matching for switch (21)

```java
// Before
String describe(Object o) {
    if (o instanceof Integer i) return "int: " + i;
    if (o instanceof String s) return "str: " + s;
    if (o == null) return "null";
    return "other";
}

// After
String describe(Object o) {
    return switch (o) {
        case Integer i -> "int: " + i;
        case String s  -> "str: " + s;
        case null      -> "null";
        default        -> "other";
    };
}
```

### Sealed classes (17)

```java
public sealed interface Result<T> permits Ok, Err {}
public record Ok<T>(T value) implements Result<T> {}
public record Err<T>(String message) implements Result<T> {}

// Compiler enforces exhaustive switch
String render(Result<String> r) {
    return switch (r) {
        case Ok<String> o -> "ok: " + o.value();
        case Err<String> e -> "err: " + e.message();
        // No default needed — sealed exhausted.
    };
}
```

Sealed + records + pattern matching = cleaner Result/Either patterns than Java 8 ever supported.

### Virtual threads (21)

Replace `ExecutorService` + thread pools for blocking I/O:

```java
// Before: thread-per-request fixed pool, blocks under load
ExecutorService exec = Executors.newFixedThreadPool(200);

// After: virtual threads — millions of lightweight threads
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();

exec.submit(() -> {
    var response = http.send(request, BodyHandlers.ofString());   // blocks
    db.save(response.body());                                     // blocks
});
```

Per-request virtual threads are cheap. Server can handle 10k+ concurrent blocking calls without thread exhaustion.

### Text blocks (15+)

```java
// Before: hell
String json = "{\n" +
    "  \"name\": \"Mahesh\",\n" +
    "  \"plan\": \"pro\"\n" +
    "}";

// After
String json = """
    {
      "name": "Mahesh",
      "plan": "pro"
    }
    """;
```

### `var` (10+)

```java
var customer = customerRepo.findById(id).orElseThrow();
var charges = chargeRepo.findByCustomerId(customer.getId());
```

Use freely for local variables when the right-hand side makes the type obvious.

## The migration steps

### 1. Update build tool

Maven `pom.xml`:
```xml
<properties>
  <maven.compiler.source>21</maven.compiler.source>
  <maven.compiler.target>21</maven.compiler.target>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

Gradle `build.gradle`:
```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

### 2. Update Spring Boot (if applicable)

Spring Boot 2.x supports up to Java 17. Spring Boot 3.x requires Java 17+.

| Spring Boot | Java | Notes |
|---|---|---|
| 2.7 | 8–17 | Last 2.x line; security patches only |
| 3.0+ | 17+ | Jakarta EE namespace change |
| 3.2+ | 17+ | Native virtual threads support |
| 3.3+ | 17+ | Recommended baseline |

Going 8 → 21 typically means Spring Boot 2.x → 3.x, which is a major upgrade (Jakarta EE namespace change: `javax.*` → `jakarta.*`). This is the single biggest pain point. Plan:

- One PR: Spring Boot 2.5 → 2.7 (catch up to last 2.x).
- One PR: Spring Boot 2.7 → 3.x with Jakarta migration.
- One PR: Java 17 → 21.

Tools:
- `OpenRewrite` recipes automate large parts of the Jakarta rename.
- `IntelliJ IDEA` "Migrate javax → jakarta" inspection.

### 3. Hibernate / JPA

Hibernate 5.x → 6.x (with Spring Boot 3). Breaking changes in dialect names, `@Type` annotation, query syntax. Read the Hibernate 6 migration guide carefully.

### 4. Other big-jumps

| Library | Notes |
|---|---|
| Lombok | Bump to latest; older versions break on 17+. |
| Mockito | 4.x → 5.x for Java 17+ ByteBuddy support. |
| JUnit 4 → 5 | Often goes alongside; `Vintage` engine bridges short-term. |
| Logback / SLF4J | Mostly fine; 1.4.x supports Java 11+. |
| Guava / Apache Commons | Bump to latest. |

### 5. Module path (optional)

Java 9 introduced JPMS (modules). Most apps don't need them — classpath is fine. Don't migrate to module path unless you have a clear need (sealed library boundaries, smaller distributions).

### 6. Reflective access warnings

Java 17+ enforces strong encapsulation. Old reflection-heavy code emits warnings. Common offenders: `cglib`, older mocking frameworks, some serialization libraries.

Escape hatch:
```
--add-opens=java.base/java.lang=ALL-UNNAMED
--add-opens=java.base/java.util=ALL-UNNAMED
```

But these are *escape hatches*. Update the offending dependency rather than carrying the flags forever.

### 7. GC defaults

Java 9+ default to G1GC; Java 17+ supports ZGC and Shenandoah. For most apps G1 is fine; for low-latency (< 10ms p99 GC pause), ZGC.

### 8. Native image (optional)

GraalVM native image works well with Spring Boot 3.x + Java 21. Cold start drops to milliseconds. But native image is its own project — don't combine with version upgrade.

## CI / build updates

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    java: [21]                          # was 8
steps:
  - uses: actions/setup-java@v4
    with:
      distribution: temurin             # Eclipse Temurin (formerly AdoptOpenJDK)
      java-version: ${{ matrix.java }}
      cache: maven
```

Docker:
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
# ... build steps
FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/*.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## Anti-patterns

- **8 → 21 in one PR** — too many breakages stacked. Go 8 → 17 → 21.
- **Combining Spring Boot 2 → 3 with Java upgrade in one PR** — Jakarta migration alone is risky enough.
- **`--add-opens` everywhere as a long-term solution** — bumps the technical debt; fix the underlying dep.
- **Not using records for new DTOs** — leaving boilerplate after the upgrade defeats the point.
- **`var` for non-local variables** (fields, return types) — Java doesn't allow it, but shorthand culture leaks. Locals only.
- **Migrating to virtual threads everywhere immediately** — they're great for blocking I/O, no help for CPU-bound. Audit before sprinkling.
- **Skipping Mockito update** — most painful test failure: Mockito 4 + Java 17+ ByteBuddy mismatch.
- **Ignoring deprecation warnings** — they become errors in future versions.
- **Lombok pinned to old version** — breaks unpredictably on JDK upgrade.
- **Not testing on production-like JVM args** — `-XX:+UseG1GC` etc. matters.

## Verify it worked

- [ ] `java -version` reports 21 on dev machines and CI.
- [ ] `mvn -v` / `gradle -v` report Java 21 toolchain.
- [ ] Production Docker image runs Java 21.
- [ ] All tests pass; no `--add-opens` flags in production beyond a documented short-term list.
- [ ] At least one new module uses records / pattern matching idiomatically.
- [ ] Spring Boot version is 3.3+ (if Spring app).
- [ ] Hibernate 6.x; JPA queries verified.
- [ ] Mockito ≥ 5.x.
- [ ] No deprecation warnings on build.
- [ ] Heap and GC behavior verified in load test (no regression vs. Java 8 baseline).
- [ ] If using virtual threads, a workload that previously bottlenecked on blocking I/O is materially better.
