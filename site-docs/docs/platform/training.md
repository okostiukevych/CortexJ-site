# Training & PEFT

The Training track (spec P4 + P6) is a unified JVM training API over pluggable
backends, with LoRA as the practical adaptation path and standard safetensors as
the interop container (spec P5). Modules: `cortexj-train-api`,
`cortexj-train-peft`, backend modules (`cortexj-train-djl`, `cortexj-train-remote`)
and `cortexj-train-multinode` (spec P10).

## The training contract

```java
public interface Trainer {
    TrainingPlan plan();
    TrainingResult fit(TrainingPlan plan);
}
```

`TrainingPlan` composes everything a run needs:

| Component | Role |
|---|---|
| `TrainingRequest` | model, dataset, strategy (`LORA` / `QLORA` / `FULL`), epochs, learning rate, device, free-form parameters |
| `Optimizer` | SGD (with momentum) or AdamW |
| `LossFunction` | cross-entropy, mean squared error |
| `LearningRateSchedule` | constant, warmup-decay |
| `CheckpointPolicy` | every-N-steps with retention |
| `EvaluationPolicy` | early stopping, cadence |
| `MixedPrecisionPolicy` | where the backend supports it |
| `TrainingData` | the batches (from [Data](data.md)) |

Standard implementations live in `io.cortexj.train.standard`:

```java
TrainingPlan plan = new TrainingPlan(
        TrainingRequest.builder()
                .model("company/model")
                .dataset("fingerprint-or-path")
                .strategy(TrainingStrategy.LORA)
                .epochs(3)
                .learningRate(2e-4)
                .build(),
        Standard.adamW(2e-4),
        Loss.crossEntropy(),
        Standard.warmupDecay(100, 10_000, 2e-4),
        Standard.checkpointEvery(500, 2),
        Standard.earlyStopping(3),
        MixedPrecisionPolicy.OFF,
        Device.CPU);
```

`TrainingResult` carries the artifact path, captured metrics and a provenance
holder (dataset fingerprint, seeds, checksums) — the same shape
[Experiments](experiments.md) store.

## LoRA (PEFT)

`cortexj-train-peft` implements LoRA with merge math that never touches the base
weights:

```java
// 1. declare the config: rank 2, alpha 4 (scaling 2.0), no dropout, one target
LoRAConfig config = new LoRAConfig(2, 4, 0.0, List.of("q_proj"));

// 2. hold the matrices: A is rank x input, B is output x rank
float[][] a = {{0.5f, -0.25f}, {1.0f, 0.0f}};
float[][] b = {{1.0f, 0.0f}, {0.25f, 0.5f}, {0.0f, 2.0f}};
LoRAAdapter adapter = new LoRAAdapter("q_proj", 2, 3, config, a, b);

// 3. merge: W' = W + (alpha/rank)·B·A — the base matrix stays untouched
float[][] merged = adapter.merge(baseWeights);
```

- Shape mismatches fail typed — never silently broadcast.
- The merge is the exact adapter math; `merge` is pure and repeatable.

## Safetensors interop (spec P5)

```java
// ship it: standard container, readable by any compliant parser
LoRASafetensors.write(adapter, Path.of("adapter_model.safetensors"),
        Map.of("note", "my first adapter"));

// read it back (also from foreign tooling's output)
LoRAAdapter read = LoRASafetensors.read(Path.of("adapter_model.safetensors"));
```

- The container is ordinary safetensors: `adapter.A` / `adapter.B` tensors plus
    a `__metadata__` sidecar.
- Foreign adapter naming fails **typed with a clear message**, never silently
    (`LoRASafetensorsInteropTest` pins the contract).
- GGUF-side merge, dequant/requant converters and the
    [Safetensors→GGUF converter](../converters.md#safetensors-gguf) complete the
    deployment path.

## Live fine-tuning

The full path (DJL PyTorch engine → trained delta → safetensors container →
checksum → provenance) is exercised by `DjlLiveTrainingTest`, which **skips
loudly** without the engine artifacts — the API surface and LoRA math are the
verified 0.x scope, the live engine run is opt-in. No PyTorch-replacement
claims: the goal is interop (spec G-013).

## Distributed training (spec P10)

`cortexj-train-multinode` implements data-parallel scale-out over ZeroMQ:

- **Collectives** — all-reduce and friends with the same semantics across nodes
    (`CollectivesTest`);
- **GradientWorker** — per-node gradient synchronization;
- **MultiNodeDataParallel** — the data-parallel driver (`MultiNodeDataParallelTest`);
- **TopologyPlanner** — plans the communication topology.

Tensor and pipeline parallelism are future scope pending an ADR (spec marks
them "later").

## The full chain

Dataset → fingerprint → experiment → LoRA → merge → provenance → model lockfile
→ Runtime deployment: the `cortexj-example-north-star` demo runs this
deterministically on CPU — see [the platform overview](index.md).

## Next steps

- [Experiments](experiments.md) — capture training runs reproducibly.
- [Tensor & Autograd](tensor.md) — the engine-neutral math underneath.
- [The Runtime](../index.md) — where trained adapters deploy.
