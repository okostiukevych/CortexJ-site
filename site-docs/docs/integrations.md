# Integrations

CortexJ plugs into the frameworks you already use. All adapters are optional modules —
the direct API stays the source of truth.

## Spring AI

`cortexj-spring-ai` exposes a loaded CortexJ model as a Spring AI 1.0.x `ChatModel`,
`StreamingChatModel` and `EmbeddingModel`. Requires Java 17+.

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-spring-ai</artifactId>
    <version>0.1.0-SNAPSHOT</version>
</dependency>
```

Blocking chat:

```java
try (CortexJChatModel chat = new CortexJChatModel("Qwen/Qwen3-8B")) {
    ChatResponse response = chat.call(new Prompt(List.of(new UserMessage("hi"))));
    System.out.println(response.getResult().getOutput().getText());
}
```

Streaming chat (reactor):

```java
try (CortexJStreamingChatModel chat = new CortexJStreamingChatModel("Qwen/Qwen3-8B")) {
    chat.stream(new Prompt(List.of(new UserMessage("hi"))))
        .map(r -> r.getResult().getOutput().getText())
        .subscribe(System.out::print);
}
```

Tokens flow from the CortexJ `TokenStream` into a `Flux<ChatResponse>`; cancelling the
Flux cancels the underlying subscription (backpressure-safe buffering).

Embeddings:

```java
try (CortexJEmbeddingModel embeddings = new CortexJEmbeddingModel("sentence-transformers/all-MiniLM-L6-v2")) {
    float[] vector = embeddings.embed("Java Virtual Threads");
    EmbeddingResponse response = embeddings.call(new EmbeddingRequest(List.of("a", "b"),
            EmbeddingOptionsBuilder.builder().build()));
}
```

Each adapter constructor creates (and owns) its own `CortexJ` platform by default.
Pass an existing platform to share caches, executors and event listeners:

```java
CortexJ platform = CortexJ.builder().build();
CortexJChatModel chat = new CortexJChatModel(platform, "Qwen/Qwen3-8B");   // platform NOT closed by the adapter
```

The model reference is planned by the CortexJ planner exactly like direct API usage:
ONNX/llama.cpp locally, `openai-remote` when `cortexj.remote.base.url` is configured.
Use `CortexJ.builder().allowRemoteRuntime(true)` plus environment variables to target
vLLM / llama-server / gateways.

## LangChain4j

`cortexj-langchain4j` exposes a loaded CortexJ model as LangChain4j 1.0.x `ChatModel`,
`StreamingChatModel` and `EmbeddingModel`. Requires Java 17+.

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-langchain4j</artifactId>
    <version>0.1.0-SNAPSHOT</version>
</dependency>
```

Chat:

```java
try (CortexJChatModel chat = new CortexJChatModel("Qwen/Qwen3-8B")) {
    ChatResponse response = chat.chat(
            ChatRequest.builder().messages(UserMessage.from("hi")).build());
    System.out.println(response.aiMessage().text());
    System.out.println(response.metadata().tokenUsage().inputTokenCount());
}
```

Token usage and finish reasons are mapped to LangChain4j types (`TokenUsage`,
`FinishReason`).

Streaming:

```java
try (CortexJStreamingChatModel chat = new CortexJStreamingChatModel("Qwen/Qwen3-8B")) {
    chat.chat(ChatRequest.builder().messages(UserMessage.from("hi")).build(),
            new StreamingChatResponseHandler() {
                @Override public void onPartialResponse(String token) { System.out.print(token); }
                @Override public void onCompleteResponse(ChatResponse r) { }
                @Override public void onError(Throwable t) { t.printStackTrace(); }
            });
}
```

`onPartialResponse` receives each token; `onCompleteResponse` receives the full
assistant message assembled from the stream plus usage metadata.

Embeddings:

```java
try (CortexJEmbeddingModel model = new CortexJEmbeddingModel(
        "sentence-transformers/all-MiniLM-L6-v2")) {
    Response<List<Embedding>> response =
            model.embedAll(List.of(TextSegment.from("a"), TextSegment.from("b")));
}
```

!!! note "Adapter scope"
    Multimodal/system prompts beyond the last user message are not translated by the
    adapter — use the direct CortexJ API for full `ChatMessage` control. As with the
    Spring AI adapter, the constructor owns its platform unless you pass one.

## OpenTelemetry

`cortexj-observability` maps platform events to OTel counters, histograms and spans.
Duration and token metrics are emitted as platform events and surfaced through the OTel
histogram:

- `PLAN_CREATED`
- `DOWNLOAD_COMPLETED`
- `INFERENCE_COMPLETED`

plus first-token latency and queue wait. Events carry metadata only — no prompt or
response content (see [Configuration](concepts/configuration.md#tokens-and-credentials)).
