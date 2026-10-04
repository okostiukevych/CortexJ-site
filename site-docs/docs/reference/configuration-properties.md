# Configuration properties

Every verified setting in one table. Priority chain:
**programmatic builder > system properties (`cortexj.*`) > environment variables (`CORTEXJ_*`)**.

## Settings

| Setting | System property | Environment | Builder | Notes |
|---|---|---|---|---|
| Offline mode | `cortexj.offline=true` | `CORTEXJ_OFFLINE=1` | `.offline(true)` | Uncached + offline fails with `CORTEXJ-MODEL-1003`; remote candidates excluded |
| Remote base URL | `cortexj.remote.base.url` | `CORTEXJ_REMOTE_BASE_URL` | — | OpenAI-compatible endpoints (vLLM, llama-server, gateways) |
| Remote API key | — | `CORTEXJ_REMOTE_API_KEY` | `.tokenSupplier(...)` | Never logged |
| Hugging Face token | — | `CORTEXJ_HF_TOKEN` (also `HF_TOKEN` / `HUGGING_FACE_HUB_TOKEN`) | `.tokenSupplier(...)` | Never logged |
| llama-server binary | `cortexj.llamacpp.server` | `CORTEXJ_LLAMACPP_SERVER` | — | Standard install prefixes probed as fallback |
| llama.cpp JNI lib | — | `CORTEXJ_LLAMACPP_LIB` | — | Future JNI bridge |
| Trust policy | — | — | `.trustPolicy(TrustPolicy.*)` | `ALLOW_UNSIGNED_WITH_WARNING` is the default; see [trust policies](../production.md#trust-policies) |
| Execution policy | — | — | `.executionPolicy(...)` | Soft scoring / rejection; never overrides hard constraints |
| Executor | — | — | `.executor(...)` | Injected `ExecutorService`, closed with the platform; no uncontrolled threads |
| Allow remote runtime | — | — | `.allowRemoteRuntime(true)` | Required for the `openai-remote` provider |
| Max artifact bytes | `cortexj.download.max.bytes` | `CORTEXJ_MAX_ARTIFACT_BYTES` | `.maxArtifactBytes(long)` | Enforced **inside** the byte-stream copy loop — a server lying about `Content-Length` cannot stream past the cap |
| Download pool | `cortexj.download.width`, `cortexj.download.queue` | — | — | Dedicated download executor (separate from inference async); saturation rejects `loadAsync` with a typed error |
| Load await timeout | `cortexj.load.awaitTimeoutMillis` | — | — | Bounds waiting on a concurrent download of the same artifact (default 10 min) |
| Enterprise policy file | `cortexj.enterprise.policy` | — | — | Explicit path override; default is `<CORTEXJ_HOME>/enterprise-policy.properties` auto-applied when present (see [enterprise runtime policy](../production.md#enterprise-runtime-policy)) |
| Benchmark hints file | `cortexj.benchmarks.file` | — | — | Explicit path override; default is `<CORTEXJ_HOME>/benchmarks.properties` (see [benchmark hints](../production.md#benchmark-hints)) |
| Server bind | `cortexj.server.bind` | — | — | Non-loopback bind without auth fails fast unless `cortexj.server.insecureBind=true` |
| Server auth | — | `CORTEXJ_SERVER_AUTH` | — | Bearer token; constant-time compare. CLI `--auth` is deprecated (token on the command line is visible in `ps`) |
| Server stream await | `cortexj.server.streamAwaitMillis` | — | — | Ceiling for holding an SSE exchange open; the upstream generation is cancelled on timeout/disconnect |
| Worker classpath / java | `cortexj.worker.classpath`, `cortexj.worker.java` | — | — | Overrides for the isolated-worker child spawn (defaults: `java.class.path` + `java.home`) |
| Worker request timeout | `cortexj.worker.request.timeout.millis` | — | — | Blocking request ceiling for the isolated worker channel (default 5 min) |

## Locations

- Platform cache: `~/.cortexj/cache` (checksum-verified, atomic writes,
    pinned/age-cap/manual eviction — see
    [Models & repositories](../concepts/models.md#the-cache)).
- Starter registration files: `META-INF/services/io.cortexj.spi.*` (see
    [Provider SPI](../providers.md)).

## Notes

- Credentials never appear in logs, error messages or plan explanations; prompts and
    responses are not logged by default.
- A starter/JVM mismatch fails fast with `CORTEXJ-CONF-0002` and the exact
    remediation — this is a bootstrap check, not a runtime setting.
