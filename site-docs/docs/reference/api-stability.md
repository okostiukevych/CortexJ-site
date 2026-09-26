# API stability

CortexJ follows semantic versioning. The contract surface is split into four tiers,
with the boundary between them enforced by JPMS module boundaries and the `jpms`
build profile (multi-release module-info jars).

## Stability tiers

| Tier | Packages | Stability guarantee |
|---|---|---|
| **Public API** | `io.cortexj`, `io.cortexj.model`, `io.cortexj.runtime`, `io.cortexj.planner`, `io.cortexj.hardware`, `io.cortexj.stream`, `io.cortexj.diagnostics`, `io.cortexj.error` | Broken only in major releases; Java 8 bytecode baseline; no records in shared signatures |
| **Stable SPI** | `io.cortexj.spi.runtime`, `io.cortexj.spi.repository`, `io.cortexj.spi.policy`, `io.cortexj.spi.platform`, `io.cortexj.spi.artifact` | Additive evolution; existing provider contracts keep compiling (TCK-verified) |
| **Experimental SPI** | `io.cortexj.spi.conversion` | May evolve during 1.0.x; converter `evaluate()` is format-based by design |
| **Internal** | `io.cortexj.*.internal`, `io.cortexj.core.*` internals, everything not exported in module-info | No guarantee whatsoever; not on the module path exports |

Experimental packages are explicitly marked: `io.cortexj.spi.conversion` is the only
tier-3 surface and will be promoted to Stable SPI with the first production release
that ships its converter set.

## Java 8/11 consumer protection

- Shared modules compile to Java 8 bytecode (`--release 8`), verified in CI on real
    JDK 8 (ubuntu/macos/windows).
- The public API uses only Java 8 types: `CompletionStage` for async, the own
    `TokenStream` contract for streaming, classes instead of records.
- `Flow.Publisher` / `Flux` adapters live in Java 11+/17+ modules
    (`cortexj-streaming-flow`, `cortexj-spring-ai`, `cortexj-langchain4j`) and never
    leak into shared signatures.

## Binary compatibility checks in CI

The `compat` Maven profile runs `japicmp` binary-compatibility checks against a
published baseline once a non-snapshot release exists:

```bash
mvn -Pcompat verify -Dcortexj.api.baseline=1.0.0
```

The profile is skipped when no baseline version is configured (there is no released
version yet). After 1.0.0 ships, CI fails on any binary-breaking change to the
Public API tier in minor/patch releases.

## What may change

- Internal packages and implementation details (module-info exports keep them
    encapsulated).
- Experimental SPI until promoted.
- Anything behind JDK profiles (platform modules) as long as the shared contract and
    the `PlatformModule` SPI stay stable.
