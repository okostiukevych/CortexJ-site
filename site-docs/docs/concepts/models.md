# Models & repositories

Models are addressed by a `ModelReference` and resolved by repository providers to an
**immutable revision**. Resolution is data-only — repository code is never executed —
and everything downstream (plans, cache keys, lockfiles) anchors on the resolved
revision.

## Model references

```java
ModelReference.of("Qwen/Qwen3-8B")        // Hugging Face id
```

A reference is just a string; which repository handles it is decided by provider
`supports()` checks, so custom schemes (for example `myrepo:…`) work the moment a
[custom repository provider](../providers.md#repository-provider-spi) declares them.

## Resolution and immutable revisions

Branches and tags are converted at resolution time to a value the repository will
serve byte-identically later:

- **Hugging Face** — commit hashes;
- **Plain HTTP** — ETags.

Plans, cache entries and lockfiles all reference `resolved.revision()`, so a plan made
today is the plan that runs next month.

## Built-in repositories

| Module | What it resolves |
|---|---|
| `cortexj-repository-local` | Filesystem models and `cortexj-model.json` manifests; air-gap bundles are local repositories too |
| `cortexj-repository-huggingface` | Hugging Face ids with commit pinning, offline mode, `CORTEXJ_HF_TOKEN` |
| `cortexj-repository-http` | Plain `https:` / `http:` URL references with ETag revisions |

The HTTP repository always attaches a **trust note** to the resolved descriptor, and
Hugging Face marks community models — see [trust policies](../production.md#trust-policies).

## The cache

Resolved artifacts are downloaded once into `~/.cortexj/cache` and verified:

- **Checksums are verified on download** — a mismatch fails with
    `CORTEXJ-ARTIFACT-2001` and the partial file is never exposed.
- **Atomic writes** — a download that fails midway cannot poison the cache.
- **Eviction** — pinned entries are protected; age-cap and manual eviction are
    available through `cache list` / `cache clean` (see the
    [CLI reference](../cli.md)).

!!! note "Offline mode"
    Uncached artifacts fail with `CORTEXJ-MODEL-1003`
    (*not cached and offline mode is active*) plus a remediation hint. Remote runtime
    candidates are excluded entirely. See [Production](../production.md).

## Artifact safety

- Declared artifact sizes are checked **before** download; oversized artifacts fail
    before any bytes hit the disk.
- Cache keys are sanitized — `../` traversal attempts in repository, revision or
    artifact ids fail with `CORTEXJ-ARTIFACT-2005`.

## Next steps

- [The execution planner](planner.md) — what happens after resolution.
- [Production](../production.md) — bundles, lockfiles and trust policies.
- [Provider SPI](../providers.md) — write your own repository provider.
