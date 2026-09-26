# Converters

When no direct artifact is viable, the planner can propose a **deterministic
conversion** as part of the plan (the "Level C" path). Originals always stay
immutable; converted outputs land in the cache with a provenance manifest.

## GGUF F32 ↔ F16

The bundled `gguf-float` converter rewrites float32 tensors as float16 (and back):

```bash
java -jar cortexj-cli-0.1.0-SNAPSHOT.jar convert ./model.gguf --to f16 --out ./converted
```

Every conversion produces a provenance manifest: sha256 of the output and the
converted tensor counts. The source file is never modified.

## Safetensors → GGUF

The safetensors converter re-containerizes Safetensors checkpoints into GGUF:

- F32 / F16 / BF16 and integer dtypes are supported;
- metadata is derived from the model's `config.json`, with honest provenance;
- verified end-to-end through Level C planning — a safetensors-only model is routed
    through conversion when no direct GGUF artifact exists.

## Provenance guarantees

- **Deterministic** — the same input produces the same output, byte for byte.
- **Traceable** — outputs record where they came from (provenance manifest,
    sha256 of the output, converted tensor counts).
- **Immutable originals** — source artifacts are never rewritten; conversions are
    materialized into the [cache](concepts/models.md#the-cache).

## Next steps

- [The execution planner](concepts/planner.md) — how conversion steps enter a plan.
- [CLI reference](cli.md#conversion-and-serving) — the `convert` command.
