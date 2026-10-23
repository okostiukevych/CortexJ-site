# Error codes

Every failure CortexJ raises is a typed `CortexJException` carrying a stable
code, a human message and a **remediation** list. Code prefixes map to
subsystems; the numeric part is stable within a minor release.

```java
try {
    platform.models().load(request);
} catch (CortexJException e) {
    log.error("{}: {} (remediation: {})", e.code(), e.getMessage(), e.remediation());
}
```

## CONF — bootstrap and configuration

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-CONF-0001` | required bootstrap component missing | add the starter/runtime artifact to the classpath |
| `CORTEXJ-CONF-0002` | starter/JVM mismatch (e.g. Java 25 starter on Java 8) | use the starter matching the running JVM — the message names it |
| `CORTEXJ-CONF-0003` | invalid configuration (unreadable/corrupt enterprise policy file, unsupported adapter version, invalid property value) | fix the file/permissions/syntax, or replace the adapter with a supported version — the message names the exact value |
| `CORTEXJ-CONF-0004` | no I/O executor available | pass `CortexJBuilder.executor(...)` or use the default platform |

## MODEL — resolution

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-MODEL-1001` | repository failed to resolve the reference | check the model id and network; the message carries the repository's reason |
| `CORTEXJ-MODEL-1002` | model not found | check the reference (`owner/name`); 404 from the repository |
| `CORTEXJ-MODEL-1003` | not cached and offline mode is active | `cortexj pull <model>` first, or disable offline mode |
| `CORTEXJ-MODEL-1004` | repository did not report an immutable revision | retry, or pin an explicit revision |

## ARTIFACT — download and integrity

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-ARTIFACT-2001` | download failed (network/HTTP) after retries | check connectivity; the partial is kept for resume |
| `CORTEXJ-ARTIFACT-2002` | artifact missing from the resolved model | re-plan; the repository content changed |
| `CORTEXJ-ARTIFACT-2003` | **checksum mismatch** | the source is corrupted or tampered with — verify the repository, re-download |
| `CORTEXJ-ARTIFACT-2004` | artifact over the size cap | raise `cortexj.download.max.bytes` or pick a smaller variant |
| `CORTEXJ-ARTIFACT-2005` | path traversal in repository/revision/artifact id | hostile input — never construct ids from untrusted strings |

## RUNTIME — provider state

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-RUNTIME-3001` | runtime not registered | add the `cortexj-runtime-*` artifact or register the provider |
| `CORTEXJ-RUNTIME-3002` | runtime installed but its native parts are missing | see the remediation (e.g. install `llama-server`) |
| `CORTEXJ-RUNTIME-3003` | runtime misconfigured (probe: `MISCONFIGURED`) | fix the configuration the message names (e.g. `CORTEXJ_REMOTE_BASE_URL`) |
| `CORTEXJ-RUNTIME-3004` | runtime needs a newer JVM | run on the required feature version or drop the runtime |
| `CORTEXJ-RUNTIME-3005` | runtime failed to load the model | see the cause; the planner walks the fallback chain automatically |
| `CORTEXJ-RUNTIME-3006` | the loaded model does not offer the requested capability | use a capability the model declares (`capabilities()` lists them) |

## PLAN — the planner

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-PLAN-4001` | no viable execution path | read the `rejected:` details — every candidate explains why it lost; `runtimeHealth` names what is down |
| `CORTEXJ-PLAN-4002` | pinned plan impossible (lockfile not reproducible) | restore the pinned runtime/artifact or regenerate the lockfile explicitly |
| `CORTEXJ-PLAN-4003` | estimate exceeds the configured memory limit | pick a smaller quantization or raise `cortexj.memory.limit.bytes` |

## MEM / NATIVE

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-MEM-5001` | model no longer fits available memory (load-time re-check) | free memory, smaller quantization, explicit memory limit |
| `CORTEXJ-NATIVE-6001` | native library missing | install it; the remediation names the env var to point at it |
| `CORTEXJ-NATIVE-6002` | native library present but failed to load | check architecture/versions; the cause carries the loader error |

## CONV — converters

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-CONV-7001` | conversion not supported for this format pair | the message says why (e.g. `gguf-float` only converts GGUF→GGUF) |
| `CORTEXJ-CONV-7002` | conversion failed | e.g. no convertible F32 weight tensors (quantized model) — run it as is, or convert a full-precision checkpoint |

## LOAD — model load

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-LOAD-8001` | the runtime could not load the artifact | the cause carries the engine error (bad file, OOM, unsupported graph) |
| `CORTEXJ-LOAD-8002` | lifecycle violation (e.g. load after close) | do not reuse a closed handle/platform |

## INFER — inference

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-INFER-9001` | inference failed (also: malformed tool-call payload from an OpenAI-compatible endpoint) | the cause carries the engine error (bad input shape, tokenizer issue, dead endpoint) or the malformed provider response |
| `CORTEXJ-INFER-9002` | token streaming failed | check the SSE connection / stream pool saturation (503 surfaces here) |
| `CORTEXJ-INFER-9003` | inference cancelled (by you or a close) | expected on cancellation — not a defect |
| `CORTEXJ-INFER-9004` | task unsupported by the loaded model | request a capability the model offers |

## SEC — policy

| Code | Meaning | Typical fix |
|---|---|---|
| `CORTEXJ-SEC-10001` | trust policy blocked the model (e.g. `TRUSTED_ONLY` + `trustNote`) | verify the publisher/checksums, pin a revision, or relax the policy |
| `CORTEXJ-SEC-10002` | license not in the allowlist | add the license to `allowedLicenses` on the request or pick another model |

## Patterns

- **Fallback aggregation**: when every candidate fails, the *first* (root-cause)
    failure is the primary exception; later candidates' failures are attached via
    `getSuppressed()` — inspect them before assuming the last error is the truth.
- **Every message is remediation-first**: the `remediation()` list is meant to be
    shown to the operator verbatim; `plan` / `inspect` CLI print it automatically.
