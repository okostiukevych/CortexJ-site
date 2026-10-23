# Converters

When no direct artifact is viable, the planner can propose a **deterministic
conversion** as part of the plan. Originals always stay
immutable; converted outputs land in the cache with a provenance manifest.

## GGUF F32 ↔ F16

The bundled `gguf-float` converter narrows F32 tensors to F16 (and widens back):

```bash
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar convert ./model.gguf --to f16 --out ./converted
```

Semantics (verified against real llama.cpp):

- Narrowing applies to **2-D+ weight matrices only** — 1-D F32 tensors (norms,
    scales) stay F32, matching real F16 checkpoints; narrowing them produces files
    real llama.cpp aborts on at inference (f32/f16 binary-op mix).
- Quantized tensors (Q4_K, Q6_K, …) pass through **byte-for-byte** — block sizes
    are known exactly, so sequential copy never desyncs.
- Sources with no convertible tensors (e.g. a Q4_K_M model, whose F32 tensors are
    all 1-D norms) are **refused honestly** with `CORTEXJ-CONV-7002` instead of
    writing a broken file — run the quantized model as is.
- `general.file_type` keeps its declared type (`UINT32`) — a mismatch makes real
    llama.cpp reject the converted file.

Every conversion produces a provenance manifest: sha256 of the output and the
converted tensor counts. The source file is never modified.

## Safetensors → GGUF

The safetensors converter re-containerizes Safetensors checkpoints into GGUF:

- F32 / F16 / BF16 and integer dtypes are supported;
- metadata is derived from the model's `config.json`, with honest provenance;
- verified end-to-end through planning — a safetensors-only model is routed
    through conversion when no direct GGUF artifact exists.

## Provenance guarantees

- **Deterministic** — the same input produces the same output, byte for byte.
- **Traceable** — outputs record where they came from (provenance manifest,
    sha256 of the output, converted tensor counts).
- **Immutable originals** — source artifacts are never rewritten; conversions are
    materialized into the [cache](concepts/models.md#the-cache).

## Next steps

- [The execution planner](concepts/planner.md) — how conversion steps enter a plan.
- [CLI reference](cli.md#conversion) — the `convert` command.
