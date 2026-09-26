# Declarative API

The `cortexj-declarative` module ships a declarative interface API on top of the
direct one: you describe prompts with annotations, and a dynamic proxy turns interface
calls into model requests.

## What it is

- `@ModelPrompt` — declares the prompt template on an interface method.
- A dynamic `java.lang.reflect.Proxy` implements your interface at runtime.
- `JsonPromptDecoder` — decodes model output as JSON and validates it.

## Semantics

The contract is **prompt + validation** — not guaranteed decoding. The decoder
validates loudly: when the model's output does not satisfy the expected structure, you
get an explicit failure, never a silently mangled result.

!!! warning "Scope today"
    The declarative API currently covers prompt declaration plus output validation.
    Richer declarative behavior (tool binding, structured schema enforcement beyond
    validation) is future evolution, not part of the current contract.

## Native image

Because the proxy mechanism uses `java.lang.reflect.Proxy`, native-image builds need
reachability metadata for the annotated interfaces (see
[Production](production.md#graalvm-native-image)).

## Next steps

- [Capabilities](concepts/capabilities.md) — what the proxy builds on.
- [API stability](reference/api-stability.md) — how surface evolution is governed.
