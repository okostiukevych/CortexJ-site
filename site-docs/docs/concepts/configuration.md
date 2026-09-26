# Configuration

CortexJ is configured with a strict priority chain:

```text
programmatic builder  >  system properties (cortexj.*)  >  environment variables (CORTEXJ_*)
```

The full settings table lives in the
[configuration reference](../reference/configuration-properties.md).

## The builder

```java
CortexJ platform = CortexJ.builder()
        .offline(true)
        .allowRemoteRuntime(true)
        .trustPolicy(TrustPolicy.TRUSTED_ONLY)
        .executionPolicy(new PrivacyPolicy())
        .build();
```

The platform is `AutoCloseable` — close it to release executors and stop managed
processes. Every subsystem uses the injected `ExecutorService`
(`CortexJBuilder.executor(...)`), closed with the platform: CortexJ never creates
uncontrolled threads.

## Offline mode

Three ways to say the same thing:

```java
CortexJ platform = CortexJ.builder().offline(true).build();
```

```bash
java -Dcortexj.offline=true -jar cortexj-cli-*.jar plan <model>
```

```bash
CORTEXJ_OFFLINE=1 java -jar cortexj-cli-*.jar plan <model>
```

In offline mode remote repositories and remote runtimes are excluded entirely, and
uncached artifacts fail with `CORTEXJ-MODEL-1003` plus a remediation hint.

## Fail-fast JVM validation

Starters validate the JVM at bootstrap. A mismatch (for example the Java 25 starter
on a Java 8 JVM) fails with `CORTEXJ-CONF-0002` and the exact remediation — *replace
with `cortexj-starter-java8`* — before any of your code runs.

## Tokens and credentials

Tokens are supplied programmatically via `CortexJBuilder.tokenSupplier(...)`, or
through `CORTEXJ_REMOTE_API_KEY`, `HF_TOKEN` / `HUGGING_FACE_HUB_TOKEN` (also
`CORTEXJ_HF_TOKEN`).

!!! note "Credentials are never logged"
    Tokens never appear in logs, error messages or plan explanations. Prompts and
    responses are not logged by default either — platform events carry only metadata
    (runtime, model, artifact, durations, byte counts).

## Next steps

- [Configuration properties](../reference/configuration-properties.md) — every
    verified knob in one table.
- [Production](../production.md) — trust policies, lockfiles and bundles.
