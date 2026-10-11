# Data

The Data track (spec P2) is the single data abstraction behind Lab, Train and
evaluation: typed frames, datasets, deterministic splits and fingerprinting.
Modules: `cortexj-data-api` plus one adapter module per source.

## Frame, Column, Schema

```java
Frame frame = CsvFrame.read(Path.of("reviews.csv"));
System.out.println(frame.rowCount());          // rows
System.out.println(frame.schema().names());    // [id, stars, text]
```

- Types are **inferred** per column: `LONG`, `DOUBLE`, `BOOLEAN`, `STRING`.
- Empty cells are missing values — never silently coerced to `0` / `""`.
- `Column<T>` is typed access; `frame.schema()` describes names, order and types.

## Adapters

| Module | Reads | Notes |
|---|---|---|
| `cortexj-data-csv` | CSV files | header row + inferred types |
| `cortexj-data-parquet` | Parquet files | columnar, schema-carrying |
| `cortexj-data-jdbc` | any JDBC source | `PreparedStatement` based reads |
| `cortexj-data-tablesaw` | Tablesaw tables | interop in both directions |
| `cortexj-data-arrow` | Arrow record batches | streaming-friendly |
| `cortexj-data-json` | JSON records | line-delimited and array forms |
| `cortexj-data-tribuo` | Tribuo datasets | conversion in **both** directions (frame → Tribuo for training) |

The same logical data read through any adapter produces an identical typed frame
— adapters are interchangeable at the `Frame` boundary.

## Fingerprinting — the dataset identity

```java
String fingerprint = Fingerprint.ofFrame(frame);
```

The fingerprint is content-derived, **not source-derived**: the same data read
through CSV, Parquet, Tablesaw or Arrow yields the same fingerprint
(cross-checked in the adapter tests). This is what makes it the dataset
identity:

- [Experiments](experiments.md) capture it as dataset provenance;
- the [planner](../concepts/planner.md) never sees it — it is a platform-track
    concept, not a runtime one;
- two runs whose fingerprints match trained on byte-identical data.

## FrameDataset: iteration, batches, splits

```java
FrameDataset dataset = new FrameDataset(frame);

dataset.count();                                   // row count
for (List<String> row : dataset) { ... }           // row iteration

List<Dataset<List<String>>> batches = dataset.batches(32);  // mini-batches

Split<Dataset<List<String>>, Dataset<List<String>>> split =
        dataset.split(0.8, 42L);                   // train/test
```

- `batches(n)` cuts the frame into row blocks — the input shape
    [training](training.md) consumes.
- `split(fraction, seed)` is **deterministic by seed**: the same seed on the
    same frame produces the same split on every machine and every JVM (asserted
    by tests).
- `StandardDatasets.blobs(...)` / `StandardDatasets.linear(...)` generate small
    synthetic frames for tests and tutorials.

## Writing your own source

Implement `DataSource` (→ `Frame`) for a custom store. The contract:

1. Produce a `Frame` with an explicit or inferred `Schema`;
2. never execute code embedded in the data (the platform rule everywhere);
3. fingerprint comes from content — you do not need to provide one.

Register nothing: adapters are plain libraries, used directly. The
[Tribuo adapter](#adapters) is the reference for two-directional conversion.

## Next steps

- [Experiments](experiments.md) — anchor a run on the fingerprint.
- [Training & PEFT](training.md) — feed training plans from datasets.
- [Lab & Notebook](lab.md) — explore frames interactively.
