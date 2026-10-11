# Runtime options

Runtime-specific options travel on the execution request and reach only the
runtime they name — no global flag surface, no cross-runtime leakage.

## Passing options

```java
ExecutionRequest request = ExecutionRequest.builder(ModelReference.of("./model.gguf"))
        .option("llamacpp", "gpuLayers", 99)     // runtime id -> option -> value
        .build();

try (ModelHandle model = platform.models().load(request)) {
    // the planner passes {"llamacpp": {"gpuLayers": 99}} to that runtime only
}
```

Unknown options are rejected by the target runtime with a typed error — options
are allowlisted per provider, never silently ignored.

## llamacpp

| Option | Type | Meaning |
|---|---|---|
| `gpuLayers` | int | maps to `--n-gpu-layers` (`99` = offload everything on Metal/CUDA). Anything but this option is rejected — the provider does not guess server flags |

Notes:

- The option is part of the shared-endpoint cache key — a CPU server and a GPU
    server are never mixed.
- Omitted means `auto` (llama-server decides).
- The binary is discovered via `CORTEXJ_LLAMACPP_SERVER`, `cortexj.llamacpp.server`
    or standard install paths (see [runtimes](../concepts/runtimes.md#llamacpp)).

## openai-remote

| Option | Type | Meaning |
|---|---|---|
| `model` | String | override the served model id (defaults to the artifact's `remoteModel` metadata, then the descriptor's canonical id) |
| `enableRerank` | boolean | enables `/v1/rerank` calls for reranking tasks (rejected with a remediation hint when a rerank request arrives without it) |

Credentials never travel as options: `CORTEXJ_REMOTE_API_KEY` /
`.remoteApiKey(...)` are delivered sealed — only the built-in remote runtime
sees them.

## onnx / djl

No user-facing options today — the providers derive everything (auxiliary
inputs, tokenizer) from the model directory. Embedding request caps are
platform-level, not runtime options: see
[capabilities](../concepts/capabilities.md#request-caps).

## Related pages

- [The execution planner](../concepts/planner.md) — how options flow into the plan.
- [Error codes](error-codes.md) — the typed refusals above.
