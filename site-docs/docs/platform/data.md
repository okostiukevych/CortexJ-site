# Data

The Data track (spec P2) is the single data abstraction behind Lab, Train and
evaluation: typed frames, datasets, deterministic splits and fingerprinting.

## Frame and Schema

`cortexj-data-api` defines `Frame`, `Column`, `Schema` and `ColType`. Types are
inferred (`LONG` / `DOUBLE` / `BOOLEAN` / `STRING`), empty cells are missing
values:

```java
Frame frame = CsvFrame.read(Path.of("reviews.csv"));
System.out.println(frame.rowCount());
System.out.println(frame.schema().names());    // [id, stars, text]
```

## Adapters

| Module | Reads |
|---|---|
| `cortexj-data-csv` | CSV files |
| `cortexj-data-parquet` | Parquet files |
| `cortexj-data-jdbc` | any JDBC source |
| `cortexj-data-tablesaw` | Tablesaw tables |
| `cortexj-data-arrow` | Arrow record batches |
| `cortexj-data-json` | JSON records |
| `cortexj-data-tribuo` | Tribuo datasets (both directions) |

## Fingerprinting — the dataset identity

```java
String fingerprint = Fingerprint.ofFrame(frame);
```

The same data read through CSV, Parquet, Tablesaw or Arrow yields the **same
fingerprint** (cross-checked in the adapter tests). The fingerprint is what
[experiments](experiments.md) capture as dataset provenance and what makes a run
reproducible.

## Datasets and splits

```java
FrameDataset dataset = new FrameDataset(frame);
Split<Dataset<List<String>>, Dataset<List<String>>> split =
        dataset.split(0.8, 42L);   // same seed -> same split, always
```

Splits are deterministic by seed. `StandardDatasets` ships small synthetic
generators (blobs, linear) for tests and tutorials.

## Next steps

- [Experiments](experiments.md) — anchor a run on the fingerprint.
- [Training & PEFT](training.md) — feed training plans from datasets.
