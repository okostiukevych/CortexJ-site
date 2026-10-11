# The CortexJ platform

CortexJ Runtime (this site's main section) is the first production product of a
larger platform. Spec v3.0 defines the long-term tracks P1–P11 — an end-to-end JVM
AI environment from experimentation to production. Several of those tracks are
already implemented in the repository and verified (JShell lab, data frames,
experiments, LoRA/PEFT, tensor + autograd, distributed collectives, research
cards) — they ship as **incubating `0.x` modules**.

!!! warning "Two-level messaging, deliberately"
    The Runtime is the production product: Java 8 baseline, TCK-verified
    providers, live-verified against real models. The platform tracks are
    incubating: APIs may evolve, live training on real engines is gated, and no
    PyTorch-replacement claims are made. Production dependencies should anchor on
    the Runtime modules; incubating modules are for experimentation.

## The tracks

| Track | Spec | Modules today | Maturity |
|---|---|---|---|
| Runtime / inference | P0 | `cortexj-api` … `cortexj-server`, CLI | beta (1.0 DoD recorded) |
| Lab / Notebook | P1 | `cortexj-lab-api` (JShell session), `cortexj-notebook` (Jupyter kernel) | incubating |
| Data | P2 | `cortexj-data-api` + CSV / Parquet / JDBC / Tablesaw / Arrow / JSON / Tribuo adapters | incubating |
| Experiments | P3 | `cortexj-experiments-api`, `cortexj-experiments-local` | incubating |
| Training & PEFT | P4 / P6 | `cortexj-train-api`, `cortexj-train-peft` (LoRA), DJL/remote backends | incubating |
| PyTorch interop | P5 | `cortexj-convert-safetensors`, LoRA safetensors, optional LibTorch backend | incubating |
| Tensor | P7 | `cortexj-tensor-api`, CPU reference + Vector backends, optional LibTorch | incubating |
| Autograd | P8 | `cortexj-autograd` (tape graph, differential tests) | incubating |
| NN modules | P9 | `cortexj-nn` (Linear, sharded linear, pipeline) | incubating |
| Distributed training | P10 | `cortexj-train-multinode` (collectives, data-parallel, topology planner) | incubating |
| Research ecosystem | P11 | `cortexj-research` (model/dataset cards, citations), benchmarks module | incubating |

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
