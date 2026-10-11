# Experiments

The Experiments track (spec P3) makes every run reproducible: parameters,
metrics, artifacts, model revision, dataset hash, JDK and seeds are captured
into a local store with embedded provenance.

## Capture a run

```java
ExperimentStore store = new LocalExperimentStore(Path.of("experiments"));
try (Experiment experiment = store.start("my-first-run", 42L, fingerprint,
        ModelIdentity.of("company/model", "rev-1"), "reference/0.1")) {
    experiment.parameter("learningRate", "2e-4");
    experiment.metric("validation.loss", "0.42");
    experiment.artifact(Path.of("adapter.safetensors")); // copied, source untouched
}
```

- **Provenance is embedded** — the record carries the model revision, dataset
    fingerprint, seeds, backend, hardware and JDK at capture time.
- **Artifacts are copied** — closing the experiment writes
    `runs/<id>/manifest.properties`, `metrics.properties` and `artifacts/`.
- **Read it back** — `store.list()` and `store.get(id)` return typed
    `ExperimentRecord`s (the double-run reproducibility proof lives in
    `LocalExperimentStoreTest`).

## What lands in the record

| Field | Source |
|---|---|
| parameters / metrics | your `experiment.parameter(...)` / `.metric(...)` calls |
| artifacts | copied into the store with checksums |
| model identity | `ModelIdentity.of(name, revision)` — the immutable revision |
| dataset fingerprint | the [Data](data.md) fingerprint |
| environment | JDK, OS, cores, captured at `start()` |
| seeds | the seed you pass to `start()` |

## Pluggable trackers

The local store is the 0.x implementation; external trackers (MLflow-style) are
a tracked follow-up — the `ExperimentStore` interface is the seam.

## Next steps

- [Training & PEFT](training.md) — feed adapter artifacts from a run into
    deployment.
- [Data](data.md) — where the fingerprint comes from.
