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
detected separately (the resolved revision must equal the locked revision). When the
lockfile pins an artifact sha256, the resolved artifact's checksum must match it — a
different binary under the same file name is rejected at plan time. The
`cortexj:verify` Maven plugin cross-checks the lockfile against the bundle manifest
when both are configured.

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

- The artifact size limit is enforced **inside** the streaming copy loop — even
    when a server lies about (or omits) `Content-Length`, the byte cap aborts the
    download before the disk fills; oversized declared sizes fail before any bytes
    are written.
- Cache keys are sanitized — `../` traversal attempts in repository/revision/artifact
    ids fail with `CORTEXJ-ARTIFACT-2005`; cache/local-repo paths are symlink-proof
    (containment via real paths).
- Bundle manifests cannot reference files outside the bundle directory, and every
    artifact entry carries an `integrity` marker (`sha256-verified` when the
    repository declared a checksum that was verified at bundle time, `unverified`
    otherwise — tampering can no longer be laundered into a "verified" bundle).
- Checksums are verified on download; mismatches fail with `CORTEXJ-ARTIFACT-2001`
    and the partial file is never exposed. HTTP Range resume is validated against
    `Content-Range` (a wrong-offset 206 restarts the download from scratch).
- In-use artifacts are **pinned** for the model lifetime: the cache never evicts a
    file a live runtime is using; the pin is released when the last handle closes.
- Downloads back off exponentially between retries, honor `Retry-After`, and keep
    the resume partial on transient failures (401/403/404 fail fast; only 416 wipes
    the partial).

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

The `llamacpp` bridge is `STATELESS` — llama-server is a concurrent HTTP server, so
concurrent generations on one `ModelHandle` overlap instead of serializing; each
generation still pays its own request. Requests inherit no environment secrets: the
spawned `llama-server` child gets an allowlisted environment (PATH/HOME/TMPDIR and
`LLAMA_*` only).

## Enterprise runtime policy

An organization can enforce a centrally supplied runtime policy without writing code:
drop `enterprise-policy.properties` into the platform working directory
(`<CORTEXJ_HOME>`, by default `~/.cortexj`) — or point
`-Dcortexj.enterprise.policy=<path>` at a file elsewhere. Every plan then applies the
policy; rejected candidates appear in the plan explanation with a clear reason.

```properties
# comma-separated runtime ids that must never be used
forbiddenRuntimes=llamacpp,djl

# false = cache/local-only operation: candidates requiring a download are rejected
allowDownloads=false

# comma-separated license whitelist; artifacts without an approved license are rejected
approvedLicenses=apache-2.0,mit

# true = artifacts without a declared checksum cannot be planned
requireVerifiedArtifacts=true

# false = non-CPU (GPU/NPU) device candidates are rejected
allowGpu=false

# false = remote runtimes are rejected (default remote set: openai-remote)
allowRemote=false
remoteRuntimes=vendor-cloud,openai-remote
```

Behavior notes:

- The file is applied automatically at platform construction; the effective policy
    file path is logged at INFO.
- An unreadable file is NOT applied and logs a WARNING (fail-open by design: an ops
    typo must not silently disable the platform) — check the logs after rollout.
- A file that declares no restrictions is ignored with a WARNING (no silent no-ops).
- The policy composes with user-supplied `ExecutionPolicy` instances from the
    builder; all of them must allow a candidate.

## Benchmark hints

Besides writing a custom `ExecutionPolicy`, a team can encode measured performance
into planning decisions with a locally supplied hints file — no code, no benchmark
learning, fully deterministic. Drop `benchmarks.properties` into the platform working
directory (or `-Dcortexj.benchmarks.file=<path>`):

```properties
# bench.<runtime>,<family>,<quantization>,<device>,<metric>=<value>
# runtime / family / quantization may be '*' (wildcard); device is cpu/gpu/npu/*

bench.llamacpp,qwen3,*,cpu,throughputTokensPerSecond=85
bench.onnx,qwen3,*,cpu,throughputTokensPerSecond=40
bench.onnx,qwen3,*,cpu,latencyMillis=180
```

- `family` matches as a lowercase substring of the model reference (`qwen3` matches
    `Qwen/Qwen3-8B`).
- Metrics: `throughputTokensPerSecond` (higher better), `latencyMillis` (lower
    better), `memoryGb` (lower better).
- Scoring is a bounded, deterministic function of the file alone: the same file
    always produces the same plan; adjustments appear in the plan explanation
    (`benchmark:throughput 85.0 t/s (+15)`).
- Malformed or unknown entries are ignored with a WARNING, never a plan failure.

## Isolated native worker

Native runtimes can crash the JVM. Worker mode provides crash isolation without
changing the public model API:

```java
// the child JVM hosts the real runtime; the parent only speaks the protocol
CortexJ platform = CortexJ.builder()
        .runtimeProvider(new IsolatedRuntimeProvider("onnx"))
        .build();
```

`IsolatedRuntimeProvider` (module `cortexj-runtime-isolated`, opt-in per runtime via
programmatic wiring) spawns `java -cp <java.class.path> io.cortexj.worker.WorkerHost
<runtimeId>` (`cortexj-worker` module) and proxies probe/open/load/execute/stream
over an NDJSON stdio protocol. A dead or hung child surfaces as typed CortexJ errors
(never a parent crash) and the next probe respawns a fresh worker; the destroy →
waitFor → destroyForcibly ladder guarantees no orphans. Streaming tokens flow through
the worker and cancellation stops the child generation. See
`docs/isolated-worker.md` in the repository for the protocol and v1 limitations.

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
