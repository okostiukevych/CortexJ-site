# Measured performance

The specification sets runtime KPI targets. This page reports **measured** numbers —
produced by harnesses in the repository, not hand-written claims. Where a number
below was measured on one machine, the harness that reproduces it is named; the
measurement contract lives in CI, the hardware-specific numbers live here.

## Model information for every run below

| Item | Value |
|---|---|
| Model | TinyLlama-1.1B-Chat, Q2_K GGUF (~460 MB) |
| Runtime | llama.cpp (`llama-server`) via `cortexj-runtime-llamacpp` |
| Workload | prompt "Once upon a time, in a land far away,", 24 tokens, greedy, seed 42 |
| Protocol | 20 measured iterations after 3 warmup (per leg) |
| Machine | macOS arm64 (Apple Silicon), local model, same JVM for both legs |
| Framework heap (no model) | 1 MB after warm-up (`PlatformOverheadTest`) |
| Model files | downloaded into a `@TempDir`, **deleted when the run ends** |

## Results (§124.1 — time to first local response)

| Metric | Measured | Target |
|---|---|---|
| Download (460 MB from Hugging Face) | **8.7 s** | — |
| Cold time-to-first-response (manifest → plan → server → first tokens) | **0.87 s** | ≤ 600 s |

## Results (§124.2 — framework overhead vs direct engine)

| Metric | Platform leg | Direct leg (llama-server HTTP) | Target |
|---|---|---|---|
| Latency p50 | 167 ms | 172 ms | platform within +2% of direct |
| Latency p95 | 175 ms | 174 ms | — |
| Throughput | 143.7 tok/s | 139.5 tok/s | degradation < 5% |

- **Framework latency overhead: −2.9%** (the platform leg measured marginally
  *faster* than the direct engine within run-to-run noise) — comfortably inside
  the `< +2%` target envelope for this machine.
- **Framework throughput degradation: −3.0%** (i.e. the platform leg was not
  slower) — inside the `< 5%` target.

- **Framework memory overhead without the model: 1 MB** (target `§124.2`:
  `< 100 MB`) — the JVM heap the fully-warmed platform (discovery, descriptors,
  planner, telemetry) itself costs. Measured by `PlatformOverheadTest`
  (`@Tag("perf")`, fake engine, heap settled after 300 warm plans).

The dispatch-level overhead that does not show up at this coarse timescale is
what the same fake-engine harness pins down: p50 = 2.6 µs, p99 = 7.3 µs of pure
platform dispatch per generate call over 1000 calls.

## How to reproduce

```bash
# full live KPI harness (downloads a ~460 MB model into a TempDir, deletes it after)
mvn -B -pl benchmarks -Pperf -Dtest=LiveRuntimeKpiTest -DfailIfNoSpecifiedTests=false test

# pure dispatch + framework-heap overhead (no model, no engine)
mvn -B -Pperf -pl core -Dtest=PlatformOverheadTest -DfailIfNoSpecifiedTests=false test
```

The report is written to `benchmarks/target/kpi-report.txt` during the run.
The live harness is **not** part of default CI (it downloads a real model);
the CI-measured KPI that runs on every lane is the
[compatibility matrix](compatibility-matrix.md) (valid-plan rate).

## Honesty notes

- Numbers are machine- and model-specific; they are a measured instance of the
  targets, not a universal constant. Re-run the harness on your own hardware
  before quoting these numbers elsewhere.
- The JVM heap delta of a live run is dominated by the engine process/model
  weights (engine-owned), not by framework allocation — the `< 100 MB framework
  overhead` clause of §124.2 targets framework-own memory and is what the
  fake-engine harness measures (1 MB, published above).
- `-Pperf` never runs in default CI (needs a real download; gated by
  `excludedGroups=perf`).

## Related

- [Compatibility matrix](compatibility-matrix.md) — the CI-measured KPI
  (valid-plan rate over a curated matrix).
- [API stability](api-stability.md) — what 1.0.0 guarantees at the contract
  level.
