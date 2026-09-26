# Capabilities

A loaded `ModelHandle` exposes typed capabilities. You ask for the interface you need;
the planner has already made sure the runtime behind the handle actually supports it.

```java
TextGenerationCapability generator =
        model.capability(TextGenerationCapability.class);
```

## Text generation

```java
GenerationResponse response = generator.generate(
        GenerationRequest.prompt("Explain MVCC briefly"));
System.out.println(response.text());
```

For one-liners, `ModelHandle` ships convenience shortcuts (`generate()` / `embed()`)
that resolve the capability and execute in one call.

## Embeddings

The [ONNX runtime](runtimes.md#onnx-runtime) serves embedding models; batches are
fully supported:

```java
try (ModelHandle model = platform.models().load(
        ModelReference.of("sentence-transformers/all-MiniLM-L6-v2"))) {
    EmbeddingCapability embedder = model.capability(EmbeddingCapability.class);
    EmbeddingResponse response = embedder.embed(
            EmbeddingRequest.texts(List.of("hello", "world")));
    float[] vector = response.vectors().get(0);
}
```

Mean pooling over the token dimension is applied automatically; tokenization goes
through the shared `HfTokenizer` (WordPiece, byte-level BPE, Unigram, whitespace
fallback).

## Streaming

CortexJ defines its own `TokenStream` contract on the Java 8 baseline, so streaming
works everywhere the core works. Providers serve it natively — the llama.cpp bridge
streams tokens from `llama-server` over SSE.

For Java 11+, the `cortexj-streaming-flow` module adapts `TokenStream` to
`java.util.concurrent.Flow.Publisher`; the Spring AI adapter exposes the same stream
as a `Flux<ChatResponse>`.

## Task support

The planner enforces task capabilities as a **hard constraint**: a runtime must
support at least one task the model offers, otherwise the candidate is rejected with
`TASK_UNSUPPORTED`. An embedding-only runtime never wins a text-generation model —
and vice versa.

## Next steps

- [Integrations](../integrations.md) — consume capabilities through Spring AI or
    LangChain4j.
- [The execution planner](planner.md) — how runtime/task matching is decided.
