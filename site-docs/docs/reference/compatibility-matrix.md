# Compatibility matrix

The [API stability tiers](api-stability.md) guarantee what does **not** break.
This page tracks what the runtime **does today**, and it is the source of the
§124.3 KPI: "planner valid-plan rate on curated compatibility matrix: ≥ 99%".

## How the number is produced

The matrix is a versioned artifact in the repository
(`planner/src/test/resources/compatibility/matrix.csv`), and the KPI harness
(`CuratedCompatibilityMatrixTest`) runs it on **every CI lane**:

- every combination the platform lists as viable must produce a plan whose
  selected runtime is one of those viable providers;
- a combination with no viable runtime (or no published artifact) must fail
  loudly — never a silent plan with an ill-fitting candidate;
- the handled rate must be ≥ 0.99, else CI fails.

Expectations are lane-aware: JInfer requires Java 25+, the LibTorch backend
requires Java 21+, so the same row yields a plan of `llamacpp` on a JDK 8 lane
and a loud failure from `libtorch` on a JDK 8 lane where nothing can serve a
safetensors artifact. The measured numbers below are **printed by CI**, not
hand-reported.

## Matrix scope today

```text
families:     Qwen3 (0.6B/8B, incl. vision), Mistral-7B, LLaMA 3.2 1B,
              BGE-M3, Nomic-Embed, Phi3-Vision, StarCoder2, Codestral,
              GPT-OSS / DeepSeek (remote), Qwen3/LLaMA 3.1 safetensors
runtimes:     llama.cpp (GGUF), ONNX (onnx), JInfer (GGUF, JDK 25+),
              remote-openai (remote), libtorch (safetensors, JDK 21+)
combinations: 20 curated rows (release-gated)
measured:     valid-plan rate ≥ 0.99 enforced by CI on all lanes
```

The matrix grows only via the repository file — a new model family or runtime
means a new row with declared viable providers, and the KPI harness settles
whether the planner agrees. If you find a real-world combination the planner
handles badly, the fix starts with adding the row to the matrix.

## Related

- [Measured performance](performance.md) — the live-runtime KPI numbers.
- [API stability](api-stability.md) — what the 0.x/1.0 labels promise.
- [Platform tracks](../platform/index.md) — the incubating tracks that sit
  behind the runtime.
