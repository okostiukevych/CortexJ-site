# CLI reference

The shaded `cortexj-cli` jar runs unchanged on Java 8 through 25 and reports the
detected JVM. It covers the whole model lifecycle without an IDE.

```bash
java -jar cli/target/cortexj-cli-*.jar <command> [options]
```

Global behavior: configuration follows the standard priority chain
(builder > `cortexj.*` properties > `CORTEXJ_*` env); failures are typed
`CORTEXJ-*` codes with remediation printed to stderr; exit code is non-zero on
failure (3 = model not executable, 2 = operational failure).

## Environment

| Command | What it does |
|---|---|
| `doctor` | Environment sanity check: JVM, detected platform module, starters, hardware, runtime health with probe messages, recommendation |
| `runtimes` | List registered runtimes: id, version, tasks, formats, devices |
| `hardware` | Print the hardware profile + detection notes |

## Models and plans

| Command | What it does |
|---|---|
| `inspect <model>` | Resolve and describe: revision, artifacts, viability; prints recommendations; exit 3 when not executable |
| `plan <model>` | Print the explainable execution plan: selected runtime, device, memory estimate, reasons, fallback chain |
| `pull <model>` | Download the planned artifacts into the cache (prints each planned download) |

`<model>` is a Hugging Face id (`Qwen/Qwen3-8B`), a local file
(`./model.gguf`) or a directory with `cortexj-model.json` (a bundle).

Plan/pull/run options:

| Option | Effect |
|---|---|
| `--runtime <id>` | require exactly this runtime (fallback disabled; impossible → `CORTEXJ-PLAN-4002`) |
| `--prefer-runtime <id>` | strong (+3.0) scoring preference — overrides the degraded-health penalty |
| `--offline` | forbid network access; uncached artifacts fail with `CORTEXJ-MODEL-1003` |
| `--limit-mb <n>` | hard memory limit in MiB (planner + load-time re-check) |
| `--lockfile <path>` | enforce the pinned plan (`PINNED_PLAN`) |
| `--lockfile-out <path>` | write the lockfile for the produced plan |

## Execution

```bash
java -jar cli/target/cortexj-cli-*.jar run <model> --prompt "Explain MVCC"
```

`run` resolves, plans, pulls if needed and executes. Generation parameters are
[runtime options](reference/runtime-options.md) on the API side.

## Reproducible deployments

```bash
# lockfiles: reproducible plans
java -jar cli/target/cortexj-cli-*.jar plan Qwen/Qwen3-8B --lockfile-out release.lock
java -jar cli/target/cortexj-cli-*.jar run Qwen/Qwen3-8B --lockfile release.lock

# air-gap bundles
java -jar cli/target/cortexj-cli-*.jar bundle create <model> --out ./bundle
java -jar cli/target/cortexj-cli-*.jar bundle verify ./bundle
```

Bundle options: `bundle create` accepts `--offline` (cache-only) — see
[Production](production.md#air-gap-bundles) for what a bundle contains and how
verification fails.

## Cache management

```bash
java -jar cli/target/cortexj-cli-*.jar cache list     # size + key per artifact, total
java -jar cli/target/cortexj-cli-*.jar cache clean    # remove everything evictable
```

The cache lives in `~/.cortexj/cache`; pinned (in-use) entries survive `clean`
(see [Models & repositories](concepts/models.md#the-cache)).

## Conversion

```bash
java -jar cli/target/cortexj-cli-*.jar convert ./model.gguf --to f16 --out ./converted
```

`--to` is `f16` or `f32`; provenance (output sha256, converted tensor counts)
prints on success. Sources with no convertible tensors are refused with
`CORTEXJ-CONV-7002` — see [Converters](converters.md#gguf-f32-f16) for the
exact semantics.

## Serving

```bash
java -jar cli/target/cortexj-cli-*.jar server <model> [--port 8080] [--bind 0.0.0.0] [--auth <token>]
```

OpenAI-compatible server over one loaded model: `/v1/models`,
`/v1/chat/completions` (blocking + SSE), `/v1/embeddings`. Binds loopback by
default; a non-loopback `--bind` requires auth or an explicit insecure-bind
acknowledgment; `--auth` warns that the token is visible in `ps` (prefer
`CORTEXJ_SERVER_AUTH`). Capacity and hardening knobs in
[Production](production.md#embedded-server). Ctrl+C drains gracefully.

## Every other knob

All settings (pools, timeouts, caps, policies) are listed in the
[configuration reference](reference/configuration-properties.md); typed failures
in the [error code reference](reference/error-codes.md).
