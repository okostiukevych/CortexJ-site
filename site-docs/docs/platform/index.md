# The CortexJ platform

CortexJ Runtime (this site's main section) is the first production product of a
larger platform: an end-to-end JVM AI environment from experimentation to
production. The tracks below are already implemented in the repository and
verified (JShell lab, data frames, experiments, LoRA/PEFT, tensor + autograd,
distributed collectives, research cards) — they ship as **incubating `0.x`
modules**.

!!! warning "Two-level messaging, deliberately"
    The Runtime is the production product: Java 8 baseline, TCK-verified
    providers, live-verified against real models. The platform tracks are
    incubating: APIs may evolve, live training on real engines is gated, and no
    PyTorch-replacement claims are made. Production dependencies should anchor on
    the Runtime modules; incubating modules are for experimentation.

## The tracks

| Track | Modules today | Maturity |
|---|---|---|---|
| Runtime / inference | `cortexj-api` … `cortexj-server`, CLI | beta |
| Lab / Notebook | `cortexj-lab-api` (JShell session), `cortexj-notebook` (Jupyter kernel) | incubating |
| Data | `cortexj-data-api` + CSV / Parquet / JDBC / Tablesaw / Arrow / JSON / Tribuo adapters | incubating |
| Experiments | `cortexj-experiments-api`, `cortexj-experiments-local` | incubating |
| Training & PEFT | `cortexj-train-api`, `cortexj-train-peft` (LoRA), DJL/remote backends | incubating |
| PyTorch interop | `cortexj-convert-safetensors`, LoRA safetensors, optional LibTorch backend | incubating |
| Tensor | `cortexj-tensor-api`, CPU reference + Vector backends, optional LibTorch | incubating |
| Autograd | `cortexj-autograd` (tape graph, differential tests) | incubating |
| NN modules | `cortexj-nn` (Linear, sharded linear, pipeline) | incubating |
| Distributed training | `cortexj-train-multinode` (collectives, data-parallel, topology planner) | incubating |
| Research ecosystem | `cortexj-research` (model/dataset cards, citations), benchmarks module | incubating |

## The reference chain

The tracks compose into one deterministic chain — the `cortexj-example-north-star`
demo runs it green on CPU-only machines:

```text
CSV dataset → typed Frame → fingerprint → batches/split
    → reproducible experiment (revision, dataset hash, seeds embedded)
    → LoRA adapter → merge (base untouched) → safetensors round-trip
    → provenance manifest → model lockfile → Runtime deployment
```

## Where to go

- [Lab & Notebook](lab.md) — interactive JVM sessions, Jupyter kernel.
- [Data](data.md) — `Frame`/`Dataset`, adapters, fingerprinting.
- [Experiments](experiments.md) — reproducible run capture.
- [Training & PEFT](training.md) — Trainer API, LoRA, distributed.
- [Tensor & Autograd](tensor.md) — engine-neutral tensors, gradients, NN modules.

The [Runtime](../index.md) documentation covers the production path these tracks
feed into.
