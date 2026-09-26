# Runtimes

Runtimes are plugins, discovered via `ServiceLoader` and described by declarative
descriptors. The planner decides from descriptors and policies — never from provider
order. Four runtimes ship today; the SPI is public and
[documented for your own](../providers.md#runtime-provider-spi).

## Built-in runtimes at a glance

| Runtime | Formats | Devices | Tasks | Streaming |
|---|---|---|---|---|
| `onnx` | ONNX | CPU (bundled CPU EP only) | EMBEDDING | no |
| `llamacpp` | GGUF | CPU / GPU | TEXT_GENERATION | yes (SSE) |
| `openai-remote` | OpenAI-compatible API | REMOTE | TEXT_GENERATION | yes (SSE) |
| `djl` | engine-dependent (honest probe) | engine-dependent | EMBEDDING (with artifacts) | engine-dependent |

## ONNX Runtime

The `onnx` provider executes ONNX embedding models locally through the bundled ONNX
Runtime (CPU execution provider) with a real Hugging Face tokenizer.

The planner picks it only when the resolved model offers an ONNX artifact and the
task is embedding. GGUF-only models are rejected with `FORMAT_UNSUPPORTED` reasons.

```java
try (ModelHandle model = platform.models().load(
        ModelReference.of("sentence-transformers/all-MiniLM-L6-v2"))) {
    EmbeddingCapability embedder = model.capability(EmbeddingCapability.class);
    EmbeddingResponse response = embedder.embed(
            EmbeddingRequest.texts(List.of("hello", "world")));
    float[] vector = response.vectors().get(0);
}
```

The shared `HfTokenizer` reads `tokenizer.json` next to the model and supports
WordPiece (BERT-style, with CLS/SEP wrapping), byte-level BPE (GPT-style), Unigram, and
a whitespace fallback when `tokenizer.json` is missing or unreadable.

Requirements: the model needs an `input_ids` (or equivalently named) int64 input. Mean
pooling over the token dimension is applied automatically.

!!! warning "Honest limitations"
    The bundled runtime ships the CPU execution provider only; GPU EP builds are a
    packaging concern. Text generation through ONNX is not declared — use llama.cpp
    or a remote runtime for that.

## llama.cpp

The `llamacpp` provider plans GGUF models and executes them locally through a
**managed `llama-server` process**: the provider starts `llama-server` with the
resolved artifact, polls its health endpoint, then serves blocking generation and SSE
token streaming through its OpenAI-compatible API. The process is shut down when the
model handle closes.

The probe finds the binary via `CORTEXJ_LLAMACPP_SERVER` /
`cortexj.llamacpp.server` (or standard install prefixes) and also checks
`CORTEXJ_LLAMACPP_LIB` for the (future) JNI bridge.

```bash
export CORTEXJ_LLAMACPP_SERVER=/opt/homebrew/bin/llama-server
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar run ./Qwen3-8B-Q4_K_M.gguf --prompt "hi"
```

An already-running `llama-server` works too — point the remote runtime at it:

```bash
llama-server -m Qwen3-8B-Q4_K_M.gguf --port 8080
export CORTEXJ_REMOTE_BASE_URL=http://localhost:8080
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar run Qwen/Qwen3-8B-GGUF --prompt "hi"
```

Planning behavior: GGUF formats on CPU/GPU; quantization is read from the filename
when the header is absent; memory estimation uses real GGUF header block counts and
sizes with an uncertainty margin. If GGUF is not viable, the planner reports every
rejected candidate with a reason.

## OpenAI-compatible remote

The `openai-remote` provider targets any OpenAI-compatible endpoint — vLLM,
`llama-server`, gateways. Remote execution requires an explicit opt-in:

```java
CortexJ platform = CortexJ.builder()
        .allowRemoteRuntime(true)
        .build();
```

Configuration: `CORTEXJ_REMOTE_BASE_URL` plus `CORTEXJ_REMOTE_API_KEY` (or
`cortexj.remote.base.url`). The provider covers vendor endpoints and is excluded
entirely in [offline mode](../production.md#offline-mode).

## DJL

The `djl` provider probes the DJL engine honestly — the engine artifacts stay
optional. When they are added, the provider serves ONNX embeddings through the shared
`HfTokenizer` pipeline.

## GraalVM / native image

- The **llama.cpp bridge is pure Java** (process + HTTP, no JNI) — the most
    native-image-friendly execution path. The `llama-server` binary must be present
    at runtime; verify with the probe.
- **ONNX** binds to the bundled runtime via JNI; native-image builds need the ONNX
    Runtime native-image configuration.
- Probe before claiming support in your own native-image builds.

## Next steps

- [Capabilities](capabilities.md) — what you call on a loaded model.
- [CLI reference](../cli.md) — `runtimes` and `doctor` probe everything above.
- [Provider SPI](../providers.md) — add your own runtime.
