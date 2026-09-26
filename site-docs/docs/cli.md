# CLI reference

The shaded `cortexj-cli` jar runs unchanged on Java 8 through 25 and reports the
detected JVM. It covers the whole model lifecycle without an IDE.

```bash
java -jar cli/target/cortexj-cli-*.jar <command>
```

## Environment

| Command | What it does |
|---|---|
| `doctor` | Environment sanity check: JVM, runtimes, hardware, cache |
| `runtimes` | Probe every registered runtime and report health |
| `hardware` | Print the discovered hardware profile |

## Models and plans

| Command | What it does |
|---|---|
| `inspect Qwen/Qwen3-8B` | Resolve and describe a model: revision, artifacts, tasks |
| `plan Qwen/Qwen3-8B` | Show the execution plan — runtime, device, memory, reasons |
| `pull Qwen/Qwen3-8B` | Download and cache the model's artifacts |

Plan variants:

```bash
java -jar cli/target/cortexj-cli-*.jar plan Qwen/Qwen3-8B --offline
java -jar cli/target/cortexj-cli-*.jar plan Qwen/Qwen3-8B --runtime llamacpp
```

See [the execution planner](concepts/planner.md) for what a plan contains.

## Execution

```bash
java -jar cli/target/cortexj-cli-*.jar run Qwen/Qwen3-8B --prompt "Explain MVCC"
```

`run` resolves, plans, pulls if needed and executes — streaming tokens to stdout.

## Reproducible deployments

```bash
# lockfiles: reproducible plans
java -jar cli/target/cortexj-cli-*.jar plan Qwen/Qwen3-8B --lockfile-out release.lock
java -jar cli/target/cortexj-cli-*.jar run Qwen/Qwen3-8B --lockfile release.lock

# air-gap bundles
java -jar cli/target/cortexj-cli-*.jar bundle create Qwen/Qwen3-8B --out ./bundle
java -jar cli/target/cortexj-cli-*.jar bundle verify ./bundle
```

Details in [Production](production.md).

## Cache management

```bash
java -jar cli/target/cortexj-cli-*.jar cache list
java -jar cli/target/cortexj-cli-*.jar cache clean
```

The cache lives in `~/.cortexj/cache`; pinned entries are protected from eviction
(see [Models & repositories](concepts/models.md#the-cache)).

## Conversion and serving

```bash
# deterministic GGUF F32 -> F16 conversion
java -jar cli/target/cortexj-cli-*.jar convert ./model.gguf --to f16 --out ./converted

# embedded OpenAI-compatible server
java -jar cli/target/cortexj-cli-*.jar server
```

See [Converters](converters.md) for provenance guarantees and
[Capabilities](concepts/capabilities.md) for the streaming contract the server uses.
