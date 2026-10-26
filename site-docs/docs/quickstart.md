# Quickstart

Three complete, runnable programs: text generation, streaming, embeddings. This
is the whole product surface for a first day.

## 1. Text generation

```java
import io.cortexj.CortexJ;
import io.cortexj.model.ModelHandle;
import io.cortexj.model.ModelReference;
import io.cortexj.inference.GenerationRequest;
import io.cortexj.inference.GenerationResponse;
import io.cortexj.inference.TextGenerationCapability;
import io.cortexj.error.CortexJException;

public class LegacyApp {
    public static void main(String[] args) {
        try (CortexJ platform = CortexJ.builder()
                .allowRemoteRuntime(true)
                .build()) {

            try (ModelHandle model = platform.models()
                    .load(ModelReference.of("Qwen/Qwen3-8B"))) {

                TextGenerationCapability generator =
                        model.capability(TextGenerationCapability.class);

                GenerationResponse response = generator.generate(
                        GenerationRequest.prompt("Explain MVCC briefly"));
                System.out.println(response.text());
                // response.finishReason() and response.usage() carry stop info and token counts
            }
        } catch (CortexJException e) {
            // typed failures: code, message, remediation steps
            System.err.println(e.code() + ": " + e.getMessage());
            e.remediation().forEach(step -> System.err.println("  -> " + step));
        }
    }
}
```

### What just happened

1. **Resolution** — the reference `Qwen/Qwen3-8B` resolves to an immutable
    revision (a Hugging Face commit hash). What you resolved is what runs.
2. **Planning** — the [execution planner](concepts/planner.md) collects candidate
    runtimes, applies hard constraints (JVM version, memory, task support,
    offline rules), scores survivors deterministically (including
    [probe health](concepts/planner.md#probe-health)) and produces a plan that
    explains every choice — including rejections.
3. **Execution** — the winning runtime executes: locally through
    [llama.cpp](concepts/runtimes.md#llamacpp) or
    [ONNX](concepts/runtimes.md#onnx-runtime), or remotely through an
    [OpenAI-compatible endpoint](concepts/runtimes.md#openai-compatible-remote).
    If the first choice cannot run, the planner walks a fallback chain — and
    tells you.
4. **Cleanup** — the model handle stops its runtime (a managed `llama-server`
    process is shut down gracefully), the platform closes executors and caches.

## 2. Token streaming

```java
GenerationRequest request = GenerationRequest.prompt("Count to five");

generator.stream(request).subscribe(new TokenConsumer() {
    @Override public void onToken(Token token) {
        System.out.print(token.text());          // tokens as they arrive
    }
    @Override public void onComplete(StreamCompletion completion) {
        System.out.println();                    // finish reason + usage
    }
    @Override public void onError(CortexJException error) {
        System.err.println(error.code() + ": " + error.getMessage());
    }
});
```

- A stream supports **exactly one subscription** — cancel it via the returned
    subscription; the upstream generation (and a managed `llama-server`
    request) stops with it.
- For Java 11+, the `cortexj-streaming-flow` module adapts the stream to
    `Flow.Publisher`; Spring AI exposes it as a `Flux<ChatResponse>`.

## 3. Embeddings

```java
try (ModelHandle model = platform.models().load(
        ModelReference.of("sentence-transformers/all-MiniLM-L6-v2"))) {
    EmbeddingCapability embedder = model.capability(EmbeddingCapability.class);

    EmbeddingResponse response = embedder.embed(
            EmbeddingRequest.texts(List.of("hello", "world")));
    float[] vector = response.vectors().get(0);   // dim 384 for MiniLM
}
```

Mean pooling is automatic; batches are supported up to the
[request caps](concepts/capabilities.md#embeddings). The platform stages the
model's `tokenizer.json` next to the weights automatically.

## The same thing via the CLI

```bash
java -jar cortexj-cli-1.0.2.jar doctor
java -jar cortexj-cli-1.0.2.jar run Qwen/Qwen3-8B --prompt "hi"
```

`doctor` checks your environment (runtimes, hardware, cache) before you depend
on it; `run` resolves, plans, pulls if needed and executes.

!!! tip "Ask before you run"
    `cortexj plan Qwen/Qwen3-8B` shows the execution plan — runtime, hardware,
    memory estimate and reasons — before anything is downloaded or started.

## When something fails

Every failure is a typed `CortexJException` with a stable code and remediation
steps — see the [error code reference](reference/error-codes.md). The two most
common first-day failures:

- `CORTEXJ-MODEL-1003` — you are offline and the model is not cached: run
    `cortexj pull <model>` first;
- `CORTEXJ-RUNTIME-3002` — a native runtime (llama.cpp) is not installed: the
    message names the env var to set, or the plan picks another runtime for you.

## Next steps

- [Models & repositories](concepts/models.md) — where models come from and how
    revisions are pinned.
- [The execution planner](concepts/planner.md) — how runtime choice is decided
    and explained.
- [Error codes](reference/error-codes.md) — every typed failure and its fix.
- [Integrations](integrations.md) — use CortexJ through Spring AI or LangChain4j.
