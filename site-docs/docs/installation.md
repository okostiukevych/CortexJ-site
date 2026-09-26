# Installation

CortexJ ships as a Maven BOM plus one starter per supported LTS line. The starter
matches your JVM at bootstrap: using a starter built for a newer JVM fails fast with
an actionable diagnostic, and the business code stays identical across all five.

## Requirements

- JDK 8, 11, 17, 21 or 25 — any supported distribution.
- For local GGUF execution you need [llama.cpp](concepts/runtimes.md#llamacpp)
    (`llama-server` binary).
- For ONNX embeddings the bundled ONNX Runtime (CPU execution provider) is enough —
    nothing extra to install.

## 1. Add the BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.cortexj</groupId>
            <artifactId>cortexj-bom</artifactId>
            <version>0.1.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 2. Pick your starter

| Starter | JVM |
|---|---|
| `cortexj-starter-java8` | 8 |
| `cortexj-starter-java11` | 11 |
| `cortexj-starter-java17` | 17 |
| `cortexj-starter-java21` | 21 |
| `cortexj-starter-java25` | 25 |

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-starter-java21</artifactId>
</dependency>
```

!!! warning "Starter/JVM mismatches fail fast"
    Running, for example, the Java 25 starter on a Java 8 JVM fails with
    `CORTEXJ-CONF-0002` and the exact remediation: *replace with
    `cortexj-starter-java8`*.

## What each LTS line changes

The shared modules (`cortexj-api`, `cortexj-core`, planner, repositories, providers)
compile to Java 8 bytecode, so the full platform boots on a real JDK 8. Everything
LTS-specific lives behind the `PlatformModule` SPI:

- **Java 8** — the complete core: model resolution, cache, planner, lockfiles,
    air-gap bundles, ONNX embeddings, OpenAI-compatible remote runtimes, streaming.
- **Java 21+** — the platform module uses virtual threads for I/O orchestration
    (downloads, remote calls) behind the same API.
- **Java 9+ extras** (`Flow.Publisher` adapter, Spring AI / LangChain4j adapters)
    live in separate JDK-profiled modules and never appear on the Java 8 classpath.

## The CLI jar

The shaded `cortexj-cli` jar runs unchanged on Java 8 through 25 and reports the
detected JVM:

```bash
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar doctor
```

See the [CLI reference](cli.md) for the full command set.

## Next steps

- [Quickstart](quickstart.md) — run your first generation.
- [Configuration](concepts/configuration.md) — builder, system properties,
    environment variables.
