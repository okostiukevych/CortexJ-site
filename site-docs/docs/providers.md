# Provider SPI

CortexJ is extendable at three points: runtimes, repositories and planner policies.
All three are `ServiceLoader` plugins with public SPI packages
(`io.cortexj.spi.*`), verified by the technology compatibility kit (`cortexj-tck`).

## Runtime provider SPI

### Declare yourself honestly

```java
public final class MyRuntimeProvider implements RuntimeProvider {

    private final RuntimeId id = RuntimeId.of("my-runtime");

    @Override
    public RuntimeId id() { return id; }

    @Override
    public RuntimeDescriptor descriptor() {
        return RuntimeDescriptor.builder(id)
                .version("1.0.0")
                .format(ArtifactFormat.GGUF)          // formats you can execute
                .device(DeviceType.CPU)               // devices you can run on
                .task(TaskCapability.TEXT_GENERATION) // tasks you support
                .minJavaFeatureVersion(11)
                .streamingSupported(true)
                .nativeRuntime(true)                  // honest!
                .threadSafety(ThreadSafety.SESSION_SCOPED)
                .maturity(RuntimeDescriptor.Maturity.BETA)
                .build();
    }
}
```

Everything in the descriptor is a planning input — an honest descriptor is what makes
the planner's explanations trustworthy.

### Probe, don't assume

`probe(context)` answers "can this runtime run HERE, right now?":

```java
@Override
public RuntimeProbeResult probe(RuntimeProbeContext context) {
    if (!nativeLibraryPresent()) {
        NativeRuntimeException failure = new NativeRuntimeException(
                ErrorCode.NATIVE_LIBRARY_MISSING,
                "my-runtime native library is not installed",
                "Install libmyruntime 1.2 and point MY_RUNTIME_LIB at it, or use the onnx runtime.");
        return RuntimeProbeResult.unavailable("native library missing", failure);
    }
    return RuntimeProbeResult.available("my-runtime 1.2 ready");
}
```

Health values are `AVAILABLE`, `DEGRADED`, `UNAVAILABLE`, `MISCONFIGURED`. The planner
caches probe results per platform instance and uses them in planning.

### Evaluate candidates

`evaluate(request)` decides whether a concrete artifact can run on a concrete device.
Every rejection must carry a `PlanReason` — rejected candidates surface verbatim in
plans and support reports:

```java
@Override
public RuntimeSupport evaluate(RuntimeEvaluationRequest request) {
    if (request.artifact() == null
            || request.artifact().format() != ArtifactFormat.GGUF) {
        return RuntimeSupport.unsupported(PlanReason.negative(
                ReasonCode.FORMAT_UNSUPPORTED, "my-runtime only executes GGUF"));
    }
    return RuntimeSupport.full(DeviceType.CPU);
}
```

### Open, and release what you take

`open` re-probes and either returns a `ModelRuntime` or throws the probe's typed
error. `ModelRuntime.close()` and `RuntimeModel.close()` must release everything
native — handles are closed by the platform or by try-with-resources.

Throw typed CortexJ exceptions with remediation — never raw runtime exceptions:

- load failures → `ModelLoadException` (`CORTEXJ-LOAD-*`)
- inference failures → `InferenceException` (`CORTEXJ-INFER-*`)
- native problems → `NativeRuntimeException` (`CORTEXJ-NATIVE-*`)

Declare `ThreadSafety` accurately: `SESSION_SCOPED` makes the platform serialize
access per handle with a lock; `THREAD_SAFE` lets callers run concurrently.

### Register

`src/main/resources/META-INF/services/io.cortexj.spi.runtime.RuntimeProvider`:

```text
com.example.MyRuntimeProvider
```

## Repository provider SPI

The repository subsystem is responsible **only** for discovery and resolution of model
artifacts — it never executes models.

```java
public final class MyRepositoryProvider implements ModelRepositoryProvider {

    @Override
    public RepositoryId id() { return RepositoryId.of("my-repo"); }

    @Override
    public boolean supports(ModelReference reference) {
        return reference.value().startsWith("myrepo:");
    }

    @Override
    public ResolvedModel resolve(ModelReference reference, ResolutionContext context) {
        if (context.isOffline() && !isCachedLocally(reference)) {
            throw new ModelResolutionException(ErrorCode.MODEL_OFFLINE_UNRESOLVED,
                    "my-repo cannot resolve " + reference.value() + " offline",
                    "Pre-download the model or configure a local mirror.");
        }
        String revision = lookupImmutableRevision(reference);
        ModelDescriptor descriptor = ModelDescriptor.builder(reference.value())
                .revision(revision)
                .task(TaskCapability.TEXT_GENERATION)
                .trustNote(null)   // set a note when YOU cannot vouch for the model
                .artifact(artifact)
                .build();
        List<ResolvedArtifact> artifacts = ...;
        return new ResolvedModel(descriptor, revision, id(), artifacts);
    }
}
```

Rules the planner depends on:

- **Immutable revision** — convert branches/tags to a value the repository will serve
    byte-identically later (HF commits; HTTP ETag). Plans, cache keys and lockfiles
    all anchor on `resolved.revision()`.
- **Honest sizes** — artifact `sizeBytes` feed memory estimation; return `0` when the
    source cannot provide sizes and the planner applies conservative defaults.
- **Checksums** — publish them when the source does; the download manager verifies and
    the cache refuses corrupted content.
- **Tokens** — use `context.tokenSupplier()` for authenticated sources; tokens are
    never logged.
- **Offline** — fail with `CORTEXJ-MODEL-1003` in offline mode; the TCK asserts this.
- **Trust notes** — set `trustNote` on content you cannot vouch for;
    `TRUSTED_ONLY` deployments will refuse it (see
    [trust policies](production.md#trust-policies)).

Artifact locations are `URI`s: `file:` for local content (no download),
`https:`/`http:` for remote (downloaded with checksum validation, size limits,
retries and progress events). `downloadHeaders()` carries per-request auth headers.

Register in
`src/main/resources/META-INF/services/io.cortexj.spi.repository.ModelRepositoryProvider`:

```text
com.example.MyRepositoryProvider
```

## Planner policies

See [The execution planner → Execution policies](concepts/planner.md#execution-policies)
for the `ExecutionPolicy` contract. Policies are pure functions of
`(candidate, context)`: unit test them directly with synthetic candidates, and
integrate by registering on a platform built with test providers.

## The TCK

```java
class MyProviderTckTest extends RuntimeProviderContract {
    // provider(), plus your provider-specific honest-behavior tests
}
```

```java
class MyRepositoryTckTest extends RepositoryProviderContract {
    // provider(), knownReference(), unknownReference()
}
```

The contracts validate descriptor sanity, probe reporting, evaluation reasons,
registration, precise `supports()`, immutable revisions, typed unknown-reference
errors and offline behavior (`CORTEXJ-MODEL-1003`).

## Existing implementations to study

- `cortexj-repository-local` — filesystem + `cortexj-model.json` manifests + bundles
- `cortexj-repository-huggingface` — HF resolution with commit pinning
- `cortexj-repository-http` — plain URL references with ETag revisions

## Next steps

- [API stability](reference/api-stability.md) — the SPI packages and their
    compatibility guarantees.
- [The execution planner](concepts/planner.md) — how descriptors and policies meet.
