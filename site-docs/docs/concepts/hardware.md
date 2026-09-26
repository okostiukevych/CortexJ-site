# Hardware & memory

The planner only proposes candidates that can actually run on *this* machine, right
now. That takes two inputs: what hardware exists, and how much memory a model needs.

## Discovery

`cortexj-hardware` builds a hardware profile through **safe probes — no shell-outs**.
The profile feeds every planning decision and is surfaced in plan explanations and
the `doctor` / `hardware` CLI commands.

Devices are classified as CPU, GPU (CUDA/Metal where present) or NPU. A runtime that
declares a device the machine does not have is filtered out before scoring — the
planner reports it as a rejected candidate rather than silently ignoring it.

## Memory estimation

Estimates come from real artifact data, not file sizes:

- **GGUF** — header fields are parsed (real block counts and tensor sizes), with an
    uncertainty margin applied on top.
- **Repository artifacts** — declared `sizeBytes` feed the estimate. A repository
    that cannot provide sizes returns `0` and the planner falls back to conservative
    defaults.
- **Quantization** is taken from the GGUF header, or from the filename when the
    header is absent.

A candidate that does not fit available memory is rejected by a **hard constraint**
— no score can override it — and the rejection appears in the plan with its numbers,
so you can see exactly why the 8B model did not win on a 4 GB machine.

## Next steps

- [The execution planner](planner.md) — how the profile is used.
- [CLI reference](../cli.md) — `cortexj hardware` prints your machine's profile.
