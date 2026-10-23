# Tool calling

Tool calling lets a model **propose** a tool invocation; you execute it and feed
the result back. CortexJ never executes tools itself — the model layer carries
the calls verbatim, and the caller owns the loop. CortexJ plus Spring AI gives
you the full agentic loop end to end; the direct API exposes the same contract.

## The contract

The proposal surface is additive — text-only consumers see nothing change:

```java
GenerationResponse response = generator.generate(
        GenerationRequest.builder()
                .message(ChatMessage.user("weather in Omsk?"))
                .tool(ToolDefinition.function("get_weather",
                        "Read the current weather for a city",
                        "{\"type\":\"object\"}"))
                .build());

if (response.finishReason() == FinishReason.TOOL_CALL) {
    for (ToolCall call : response.toolCalls()) {
        // call.id(), call.name(), call.arguments() - raw JSON, verbatim
        String result = executeYourTool(call.name(), call.arguments());
        // feed the result back, tied to the call by id:
        messages.add(ChatMessage.assistantToolCalls(response.toolCalls(), ""));
        messages.add(ChatMessage.tool(call.id(), result));
    }
}
```

A `TOOL_CALL` finish with an **empty** call list means the provider reported a
finish reason without a payload — treat that as a provider bug, never as
"no tool to run".

## Streaming

Structured calls stream as a batch: the consumer receives every batch **after the
final token and before the terminal**, so when `onComplete` reports `TOOL_CALL`
the calls are already in hand and execution can start immediately.

```java
stream.subscribe(new TokenConsumer() {
    public void onToken(Token token) { /* append text */ }

    public void onToolCalls(List<ToolCall> calls) {
        // implement only if you drive a tool loop; text-only
        // consumers never implement this
    }

    public void onComplete(StreamCompletion completion) { }
    public void onError(CortexJException error) { }
});
```

Callers that publish before subscribing get their calls replayed in order, like
tokens — no fast-producer race.

## OpenAI-compatible remote runtime

The `remote-openai` runtime is wire-complete in both directions:

- **Request**: declared tools render as the provider's `tools` array (the JSON
  schema is parsed, never a quoted string); assistant messages render the calls
  proposed in the previous round; tool results render tied to their call by
  `tool_call_id`.
- **Response**: non-streamed `message.tool_calls` parse into `ToolCall` values; a
  call without a function name fails typed naming the malformed entry
  (`CORTEXJ-INFER-9001`) — a malformed *schema* on the request side fails typed
  naming the tool (`CORTEXJ-CONF-0003`), never as a provider error.
- **Streaming**: `delta.tool_calls` fragments accumulate per index (id and name
  arrive once, `function.arguments` is appended verbatim); the assembled calls
  publish before the `tool_calls` terminal.

Local JNI runtimes (llama.cpp, the embedded server) expose the `tool_calls`
**finish reason** only — no payload yet. OpenAI-compatible endpoints are the
tool-calling surface today.

## Spring AI: the full loop

With `ToolCallingChatOptions` the adapter runs Spring's tool loop above the
model, unchanged — Spring executes your callbacks, CortexJ carries the
proposals and the conversation history:

```java
ChatModel model = new CortexJChatModel("Qwen/Qwen3-8B");

ChatResponse answer = model.call(new Prompt("weather in Omsk?",
        ToolCallingChatOptions.builder()
                .toolCallbacks(FunctionToolCallback
                        .builder("get_weather", (Map<String, Object> args) ->
                                weatherLookup(args))
                        .inputSchema("{\"type\":\"object\"}")
                        .inputType(Map.class)
                        .build())
                .build()));
```

What happens per round:

1. Your `ToolCallback`s are declared to the runtime as tool definitions.
2. The model's proposals wrap into a Spring `ChatResponse` with assistant
   `ToolCall` records — this is what Spring's `ToolCallingManager` reads.
3. The manager executes **your** callbacks; its conversation history (original
   messages + assistant echo + tool results) becomes the next round's prompt.
4. You see the final answer; rounds are bounded by Spring's default tool-manager
   limits, so a runaway tool fails with a typed error instead of hanging.

Honest boundaries:

- `stream()` **with** tools runs the same loop inline and emits a single final
  chunk — partial tool-call streaming is not modelled through the bridge.
- Providers that only implement `generate` keep tool-loop compatibility: calls
  ride the generated stream in the same position.
- The runtime never picks a tool, never validates arguments, never runs code.
