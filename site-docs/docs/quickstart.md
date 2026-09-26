# Quickstart

A complete, runnable example: load a model, generate text, close everything. This is
the whole product surface you touch for a first run.

```java
import io.cortexj.CortexJ;
import io.cortexj.model.ModelHandle;
import io.cortexj.model.ModelReference;
import io.cortexj.inference.GenerationRequest;
import io.cortexj.inference.GenerationResponse;
import io.cortexj.inference.TextGenerationCapability;

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
            }
        }
    }
}
```

## What just happened

1. **Resolution** — the reference `Qwen/Qwen3-8B` is resolved to an immutable
    revision (a Hugging Face commit hash). What you resolved is what runs.
2. **Planning** — the [execution planner](concepts/planner.md) collects candidate
    runtimes, applies hard constraints (JVM version, memory, task support, offline
    rules), scores the survivors deterministically and produces a plan that explains
    every choice — including rejections.
3. **Execution** — the winning runtime executes the request: locally through
    [llama.cpp](concepts/runtimes.md#llamacpp) or
    [ONNX](concepts/runtimes.md#onnx-runtime), or remotely through an
    [OpenAI-compatible endpoint](concepts/runtimes.md#openai-compatible-remote).
    If the first choice cannot run, the planner walks a fallback chain — and tells
    you.
4. **Cleanup** — the model handle stops its runtime (a managed `llama-server`
    process is shut down gracefully), the platform closes its executors and caches.

## The same thing via the CLI

```bash
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar doctor
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar run Qwen/Qwen3-8B --prompt "hi"
```

`doctor` checks your environment (runtimes, hardware, cache) before you depend on it;
`run` resolves, plans, pulls if needed and executes.

!!! tip "Ask before you run"
    `cortexj plan Qwen/Qwen3-8B` shows the execution plan — runtime, hardware,
    memory estimate and reasons — before anything is downloaded or started.

## Next steps

- [Models & repositories](concepts/models.md) — where models come from and how
    revisions are pinned.
- [Capabilities](concepts/capabilities.md) — text generation, embeddings and
    token streaming.
- [Integrations](integrations.md) — use CortexJ through Spring AI or LangChain4j.
