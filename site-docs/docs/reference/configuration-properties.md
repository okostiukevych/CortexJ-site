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
