# Configuration properties

Every verified setting in one table. Priority chain:
**programmatic builder > system properties (`cortexj.*`) > environment variables (`CORTEXJ_*`)**.

## Settings

| Setting | System property | Environment | Builder | Notes |
|---|---|---|---|---|
| Offline mode | `cortexj.offline=true` | `CORTEXJ_OFFLINE=1` | `.offline(true)` | Uncached + offline fails with `CORTEXJ-MODEL-1003`; remote candidates excluded |
| Remote base URL | `cortexj.remote.base.url` | `CORTEXJ_REMOTE_BASE_URL` | — | OpenAI-compatible endpoints (vLLM, llama-server, gateways) |
| Remote API key | `cortexj.remote.api.key` (warns: sysprops surface in `jcmd`/dumps) | `CORTEXJ_REMOTE_API_KEY` | `.remoteApiKey(...)` | Sealed delivery: only the built-in `openai-remote` runtime receives it, never logged |
| Hugging Face token | `cortexj.hf.token` (warns, prefer env) | `CORTEXJ_HF_TOKEN` (also `HF_TOKEN`) | `.tokenSupplier(...)` | Never logged |
| llama-server binary | `cortexj.llamacpp.server` | `CORTEXJ_LLAMACPP_SERVER` | — | Standard install prefixes probed as fallback |
| llama.cpp JNI lib | — | `CORTEXJ_LLAMACPP_LIB` | — | Future JNI bridge |
| Trust policy | — | — | `.trustPolicy(TrustPolicy.*)` | `ALLOW_UNSIGNED_WITH_WARNING` is the default; see [trust policies](../production.md#trust-policies) |
| Execution policy | — | — | `.executionPolicy(...)` | Soft scoring / rejection; never overrides hard constraints |
| Executor | — | — | `.executor(...)` | Injected `ExecutorService`, closed with the platform; no uncontrolled threads |
| Allow remote runtime | — | — | `.allowRemoteRuntime(true)` | Required for the `openai-remote` provider |
| Max artifact bytes | `cortexj.download.max.bytes` | `CORTEXJ_MAX_ARTIFACT_BYTES` | `.maxArtifactBytes(long)` | Enforced **inside** the byte-stream copy loop — a server lying about `Content-Length` cannot stream past the cap |
| Memory limit | `cortexj.memory.limit.bytes` | `CORTEXJ_MEMORY_LIMIT_BYTES` | `.memoryLimitBytes(long)` | Hard planning + load-time memory ceiling |
| Download pool | `cortexj.download.width`, `cortexj.download.queue` | — | — | Short-download executor; saturation fails queued downloads typed |
| Load pool | `cortexj.load.width`, `cortexj.load.queue` | — | — | Long native loads run here, separate from downloads — a stuck load no longer starves queued downloads (saturation rejects `loadAsync` with a typed error) |
| Async pool | `cortexj.async.width`, `cortexj.async.queue` | — | — | `executeAsync` fan-out; bounded with fail-fast rejects |
| Load await timeout | `cortexj.load.awaitTimeoutMillis` | — | — | Bounds waiting on a concurrent download of the same artifact (default 10 min) |
| Enterprise policy file | `cortexj.enterprise.policy` | — | — | Explicit path override; default is `<CORTEXJ_HOME>/enterprise-policy.properties` auto-applied when present (see [enterprise runtime policy](../production.md#enterprise-runtime-policy)) |
| Benchmark hints file | `cortexj.benchmarks.file` | — | — | Explicit path override; default is `<CORTEXJ_HOME>/benchmarks.properties` (see [benchmark hints](../production.md#benchmark-hints)) |
| Server bind | `cortexj.server.bind` | — | — | Non-loopback bind without auth fails fast unless `cortexj.server.insecureBind=true` |
| Server auth | `cortexj.server.auth` | `CORTEXJ_SERVER_AUTH` | — | Bearer token; constant-time compare, 5 failures/60 s per IP → 60 s block (429). CLI `--auth` warns (token visible in `ps`, prefer env) |
| Server stream await | `cortexj.server.streamAwaitMillis` | — | — | Ceiling for holding an SSE exchange open; the upstream generation is cancelled on timeout/disconnect |
| Server generate timeout | `cortexj.server.generateTimeoutMillis` | — | — | Ceiling for blocking unary generations (default 30 min); timeout answers 504 |
| Server stream workers | `cortexj.server.streamWorkers` | — | — | SSE pool size (default max(4, NCPU)); streams and unary have separate capacity (see [production](../production.md#embedded-server)) |
| Server max body | `cortexj.server.maxBodyBytes` | — | — | JSON body cap (default 8 MB); larger bodies get 413 |
| Probe wait | `cortexj.planner.probeWaitMillis` | — | — | Ceiling for joining an in-flight native probe (default 60 s) |
| Unknown-memory block | `cortexj.planner.blockUnknownMemory` | `CORTEXJ_PLANNER_BLOCK_UNKNOWN_MEMORY=1` | — | Opt-in fail-closed: unknown memory estimates block candidates instead of passing vacuously |
| Circuit cooldown | `cortexj.planner.circuitCooldownMillis` | — | — | Cooldown after 5 consecutive probe failures (default 60 s) |
| Private-host opt-out | `cortexj.http.allowPrivateHosts` | `CORTEXJ_HTTP_ALLOW_PRIVATE_HOSTS=1` | — | SSRF guard is default-on for repository/download/remote URLs; opt out only for intranet mirrors |
| Proxy | `cortexj.proxy.host`, `cortexj.proxy.port` | `CORTEXJ_PROXY_HOST`, `CORTEXJ_PROXY_PORT` | — | HTTP proxy for downloads and remote runtimes |
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
