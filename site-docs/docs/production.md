# Production

Everything CortexJ needs for strict deployments: no network, verified artifacts,
reproducible plans, controlled trust and observable execution.

Three levels of deployment isolation:

1. **Offline mode** — no network resolution at all.
2. **Air-gap bundles** — portable, verifiable model directories.
3. **Lockfiles** — reproducible plans for strict deployments.

## Offline mode

```java
CortexJ platform = CortexJ.builder().offline(true).build();
```

or

```bash
java -Dcortexj.offline=true -jar cortexj-cli-*.jar plan <model>
```

Uncached artifacts fail with `CORTEXJ-MODEL-1003` (*not cached and offline mode is
active*) plus remediation. Remote runtime candidates are excluded.

## Air-gap bundles

Create a bundle from a resolved model (or a local GGUF file):

```bash
java -jar cortexj-cli-*.jar bundle create Qwen/Qwen3-8B --out ./release-bundle
java -jar cortexj-cli-*.jar bundle create ./local-model.gguf --out ./release-bundle
```

The bundle directory contains:

- `cortexj-model.json` — the manifest: model id, resolved immutable revision,
    architecture, license, tasks, and per-artifact entries with **computed sha256**
    and size;
- all model artifacts (weights plus tokenizer/config files when published);
- checksums for every file.

Artifacts are re-used from the platform cache when present (`~/.cortexj/cache`).

Verify before deployment:

```bash
java -jar cortexj-cli-*.jar bundle verify ./release-bundle
```

Checks: manifest present and parseable, every artifact file exists, declared size
matches, declared sha256 matches (tampering is reported explicitly), artifact paths
stay inside the bundle (path-traversal safe). The exit code is non-zero on any issue.

Run from a bundle offline — a bundle is itself a local repository:

```bash
java -Dcortexj.offline=true -jar cortexj-cli-*.jar run ./release-bundle --prompt "hi"
```

## Lockfiles

```bash
java -jar cortexj-cli-*.jar plan Qwen/Qwen3-8B --lockfile-out release.lock
java -jar cortexj-cli-*.jar run Qwen/Qwen3-8B --lockfile release.lock
```

`--lockfile` enforces `PINNED_PLAN`: the planner must reproduce the locked runtime and
artifact or fail with `CORTEXJ-PLAN-4002` and a full explanation. Revision drift is
detected separately (the resolved revision must equal the locked revision).

## Trust policies

```java
CortexJ platform = CortexJ.builder()
        .trustPolicy(TrustPolicy.TRUSTED_ONLY)
        .build();
```

- `TRUSTED_ONLY` — loading a model whose descriptor carries a `trustNote` (plain
    HTTP served, community upload, unverified checksums) fails with
    `CORTEXJ-SEC-10001` and remediation.
- `ALLOW_UNSIGNED_WITH_WARNING` (default) — the model loads and a WARNING platform
    event is recorded.
- `CUSTOM` — decisions delegated to your
    [execution policies](concepts/planner.md#execution-policies).

Repositories set `trustNote` on descriptors they cannot vouch for — the HTTP
repository always does; Hugging Face marks community models.

## Artifact safety

- Declared size is checked before download; oversized artifacts fail before any bytes
    hit the disk.
- Cache keys are sanitized — `../` traversal attempts in repository/revision/artifact
    ids fail with `CORTEXJ-ARTIFACT-2005`.
- Bundle manifests cannot reference files outside the bundle directory.
- Checksums are verified on download; mismatches fail with `CORTEXJ-ARTIFACT-2001`
    and the partial file is never exposed.

## Credentials and privacy

- Tokens are supplied via `TokenSupplier` (`CortexJBuilder.tokenSupplier(...)`,
    `CORTEXJ_REMOTE_API_KEY`, `HF_TOKEN` / `HUGGING_FACE_HUB_TOKEN`).
- Tokens are never logged and never appear in error messages or plan explanations.
- Prompts and responses are not logged by default; platform events carry only
    metadata (runtime, model, artifact, durations, byte counts).

## Supply chain

- No repository code is ever executed — resolution is data-only.
- Every artifact is addressed by immutable revision; mutable branches are pinned at
    resolution time.
- Lockfiles and bundles pin exact revisions and sha256 checksums for later
    verification.

## Native runtimes

Native libraries are probed, not assumed: probe failures are honest and actionable.
llama.cpp is optional; the recommended path (`llama-server` + `openai-remote`)
isolates native code outside the JVM entirely.

## Observability

Platform events flow to OpenTelemetry through `cortexj-observability` —
`PLAN_CREATED`, `DOWNLOAD_COMPLETED`, `INFERENCE_COMPLETED`, first-token latency and
queue wait (see [Integrations](integrations.md#opentelemetry)).

## GraalVM / native image

Core and planner code avoid unnecessary reflection; ServiceLoader-based registration
works with native-image reachability metadata. The `cortexj-declarative` module uses
`java.lang.reflect.Proxy` and requires reachability metadata for annotated
interfaces. The [llama.cpp bridge](concepts/runtimes.md#graalvm-native-image) is the
most native-image-friendly execution path.
