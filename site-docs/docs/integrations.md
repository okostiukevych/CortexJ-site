# Integrations

CortexJ plugs into the frameworks you already use. All adapters are optional modules —
the direct API stays the source of truth.

## Spring AI

`cortexj-spring-ai` exposes a loaded CortexJ model as a Spring AI 2.0.1 `ChatModel`,
`StreamingChatModel` and `EmbeddingModel`. Requires Java 17+.

!!! note "Version floor"
    The adapter fails fast at construction on Spring AI 1.x classpaths with
    `CORTEXJ-CONF-0003` and the required minimum version — 1.x is end-of-life and
    receives no fixes.

!!! note "Repository"
    Both dependencies resolve from the [CortexJ Maven repository](installation.md#1-add-the-cortexj-repository)
    (or from JitPack under `com.github.okostiukevych.CortexJ`).

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-spring-ai</artifactId>
    <version>1.0.0-RC1</version>
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

Tool calling through the adapter (Spring runs the loop, CortexJ carries the calls)
is documented on its own page: [Tool calling](tool-calling.md).

## Spring Boot starter

`cortexj-spring-boot-starter` auto-configures the platform and an auto-wired
`ChatModel` from `cortexj.*` properties. Requires Java 17+ and Spring Boot 4.x
(the Framework 7 generation the Spring AI 2.x adapter targets).

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-spring-boot-starter</artifactId>
    <version>1.0.0-RC1</version>
</dependency>
```

```yaml
cortexj:
  model: Qwen/Qwen3-8B            # wires a ChatModel bean
  offline: false
  allow-remote-runtime: true
  trust-policy: ALLOW_UNSIGNED_WITH_WARNING
  remote:
    base-url: https://my-endpoint/v1
    api-key: sk-...
```

Inject and call — no wiring code:

```java
@Service
class AnswerService {
    private final ChatModel model;

    AnswerService(ChatModel model) { this.model = model; }

    String answer(String prompt) {
        return model.call(new Prompt(prompt)).getResult().getOutput().getText();
    }
}
```

Available properties (all optional, defaults mirror the direct builder):
`model`, `offline`, `allow-remote-runtime`, `working-directory`, `trust-policy`,
`memory-limit`, `max-artifact-bytes`, `shutdown-hook`, and `remote.base-url` /
`remote.api-key` / `remote.model-prefix`. The starter binds Spring's relaxed
names (`base-url`, `baseUrl`, `BASE_URL` all work); the dot-style properties on
the configuration-properties page (`cortexj.remote.base.url`) are the
direct-builder system-property surface.

Honest boundaries:

- Every bean is `@ConditionalOnMissingBean`: defining your own `CortexJ` platform
  or `ChatModel` disables the auto-configured one — the starter never fights your
  configuration.
- An unresolvable `cortexj.model` fails application startup with a typed
  `CORTEXJ-` diagnostic instead of failing on the first call.
- Proxy settings and event-listener registration stay with the direct builder API.

## LangChain4j

`cortexj-langchain4j` exposes a loaded CortexJ model as LangChain4j 1.22.x `ChatModel`,
`StreamingChatModel` and `EmbeddingModel`. Requires Java 17+.

```xml
<dependency>
    <groupId>io.cortexj</groupId>
    <artifactId>cortexj-langchain4j</artifactId>
    <version>1.0.0-RC1</version>
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

plus first-token latency and queue wait. Metric points carry low-cardinality
attributes only (event type, level, runtime) — model, artifact and exact byte
counts ride spans and measurements instead, so many distinct models cannot explode
the metric series count. Events carry no prompt or response content (see
[Configuration](concepts/configuration.md#tokens-and-credentials)).
