# Training & PEFT

The Training track (spec P4 + P6) provides a unified JVM training API over
pluggable backends, with LoRA as the practically useful adaptation path and
standard safetensors as the interop container (spec P5).

## The Trainer API

`cortexj-train-api` defines the contract surface: `Trainer`, `TrainingPlan`,
`Optimizer`, `LossFunction`, `LearningRateSchedule`, `CheckpointPolicy`,
`MixedPrecisionPolicy`, `EvaluationPolicy`, `TrainingListener` and the
`TrainingBackend` SPI:

```java
Trainer trainer = ...;              // from the backend provider
TrainingResult result = trainer.fit(trainingPlan);
result.metrics();                    // captured metrics
result.provenance();                 // dataset fingerprint, seeds, checksums
```

Backends ship for DJL (PyTorch engine) and remote execution; a live fine-tune
against a real engine is a tracked, honestly-gated follow-up — the API surface
and the LoRA math are the verified 0.x scope.

## LoRA (PEFT)

`cortexj-train-peft` implements LoRA with merge math that never touches the base
weights:

```java
LoRAConfig config = new LoRAConfig(2, 4, 0.0, List.of("q_proj"));
LoRAAdapter adapter = new LoRAAdapter("q_proj", 2, 3, config, a, b);

float[][] merged = adapter.merge(baseWeights); // W' = W + (alpha/rank)·B·A
```

- The base matrix stays untouched; shape mismatches fail typed.
- **Standard container** — `LoRASafetensors.write/read` round-trips the adapter
    through ordinary safetensors (`adapter.A` / `adapter.B`, `__metadata__`),
    readable by any compliant parser. Foreign adapter naming fails loudly, never
    silently.
- GGUF-side merge and dequant/requant converters round out the deployment path.

## Distributed training

`cortexj-train-multinode` (spec P10) implements single-node and multi-node
data-parallel training over ZeroMQ: collectives (all-reduce and friends), a
gradient worker, and a topology planner. Tensor/pipeline parallelism is future
scope pending an ADR.

## The full chain

Dataset → fingerprint → experiment → LoRA → merge → provenance → model lockfile:
the `cortexj-example-north-star` demo runs this end-to-end deterministically on
CPU (see [the platform overview](index.md#the-reference-chain)).

## Next steps

- [Experiments](experiments.md) — capture training runs reproducibly.
- [Tensor & Autograd](tensor.md) — the engine-neutral math underneath.
