# CortexJ

**CortexJ Runtime — portable AI execution for the JVM.**

One JVM API — compatible models — the best available runtime. Production AI from
legacy Java 8 applications to modern Java 25 systems, across interchangeable
local and remote runtimes, without mandatory Python.

The platform answers the infrastructure questions itself: where the model lives, which
revision to use, which artifacts are compatible, which runtimes are installed, what
hardware is available, whether memory suffices, what the fallback is — and *why* the
chosen execution path was selected.

!!! note "Status"
    CortexJ is `0.1.0-SNAPSHOT` and not yet on Maven Central. The documentation
    describes the working platform as it exists today.

## Why CortexJ

- **Model portability** — one API for any compatible model. Local filesystem,
    Hugging Face and plain HTTP repositories, all resolved to immutable revisions.
- **The execution planner** — deterministic scoring, hard constraints, fallback
    chains, and a full explanation for every plan. Provider order never decides.
- **Java 8 → 25** — starters for every LTS line. The same business code, a
    fail-fast starter/JVM validation, virtual threads from Java 21+.
- **Runtimes are plugins** — ONNX Runtime, llama.cpp (managed llama-server bridge),
    OpenAI-compatible remote runtimes (vLLM, llama-server, gateways) and DJL,
    discovered via `ServiceLoader`.
- **Integrations** — Spring AI, LangChain4j and OpenTelemetry adapters.
- **Production-ready** — model lockfiles, verified air-gap bundles, offline
    deployment, trust policies and observability metrics.

## Where to go next

- [Installation](installation.md) — add the BOM and the starter for your JVM.
- [Quickstart](quickstart.md) — from a dependency to your first generated token.
- [The execution planner](concepts/planner.md) — how CortexJ decides *why* a plan
    is what it is.

If you operate models rather than write Java, start with the
[CLI reference](cli.md) — `doctor`, `plan`, `pull` and `run` cover the whole
lifecycle without an IDE.
