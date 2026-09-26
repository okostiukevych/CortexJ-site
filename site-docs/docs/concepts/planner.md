# The execution planner

Loading a model produces an **execution plan** before anything is downloaded or
started. The plan is deterministic — same inputs, same plan — and it is explainable,
down to reason codes.

```text
Your code → CORTEXJ PLANNER → Runtime SPI (ServiceLoader) → CPU / GPU / NPU
```

The planner collects candidates (runtimes × artifacts × devices), filters them through
hard constraints, scores the survivors, and produces a plan that reports both the
chosen path and every rejected candidate — verbatim, with reasons.

## Hard constraints

Hard constraints cannot be bypassed by scores or policies:

- **JVM version** — the runtime must support the running Java feature version;
- **Memory** — the estimated footprint must fit;
- **Task support** — the runtime must support at least one task the model offers
    (`TASK_UNSUPPORTED` otherwise; an embedding-only runtime never wins a
    text-generation model);
- **Offline rules** — offline mode removes remote repositories and remote runtimes;
- **Required runtime** — an explicitly requested runtime (`--runtime llamacpp`)
    removes everything else from the candidate set.

## Soft scoring

Surviving candidates are scored with defined tie-breaks. The result is deterministic:
provider registration order never decides. Rejections are always reported with a
`PlanReason` — for example `FORMAT_UNSUPPORTED` for a GGUF-only model on the ONNX
runtime.

## Explanations

Every plan states why it is what it is: the chosen runtime and device, the memory
estimate, the quality trade-offs, and the reason each rejected candidate lost. There
are no mystery paths.

## Fallback

If the first choice cannot run — a missing `llama-server`, an oversized artifact, a
dead remote — the planner walks the fallback chain and reports every step. Fallback
is never hidden.

## Quality profiles

```java
ExecutionRequest.qualityProfile(QualityProfile.MEMORY_FIRST)
```

Quality profiles map to built-in scoring rules (`QualityProfileRule`); the chosen
quantization is visible in the plan explanation and never hidden.

## Execution policies

Policies customize soft scoring and rejection without touching the planner. A policy
is a pure function of a candidate and its planning context:

```java
public final class PrivacyPolicy implements ExecutionPolicy {

    @Override
    public String policyId() { return "privacy-no-remote"; }

    @Override
    public PolicyDecision evaluate(ExecutionCandidate candidate, PlanningContext context) {
        if (candidate.device() == DeviceType.REMOTE) {
            return PolicyDecision.reject(ReasonCode.USER_REQUIREMENT_CONFLICT,
                    "remote execution is forbidden by the privacy policy");
        }
        return PolicyDecision.neutral();
    }
}
```

Register programmatically:

```java
CortexJ platform = CortexJ.builder()
        .executionPolicy(new PrivacyPolicy())
        .build();
```

`PolicyDecision` options:

- `neutral()` — no opinion;
- `reject(reasonCode, message)` — the candidate is removed and reported with the reason;
- `scoreAdjustment(delta, reasonCode, message)` — a soft scoring shift.

!!! warning "Scores never override hard constraints"
    A `scoreAdjustment` can reorder candidates but can never make a memory-oversized
    or task-unsupported candidate viable.

## Lockfile enforcement

A lockfile pins a plan (`PINNED_PLAN`) and the model revision (`PINNED_MODEL`):

```bash
java -jar cortexj-cli-*.jar plan Qwen/Qwen3-8B --lockfile-out release.lock
java -jar cortexj-cli-*.jar run Qwen/Qwen3-8B --lockfile release.lock
```

Running with `--lockfile` must reproduce the locked runtime and artifact, or fail
with `CORTEXJ-PLAN-4002` and a full explanation. Revision drift is detected
separately. See [Production](../production.md#lockfiles).

## Ask first

```bash
java -jar cortexj-cli-*.jar plan Qwen/Qwen3-8B --offline
```

`plan` answers what will happen — runtime, device, memory, reasons — before anything
is downloaded or started.

## Next steps

- [Runtimes](runtimes.md) — what the planner chooses between.
- [Hardware & memory](hardware.md) — where the estimates come from.
- [Provider SPI](../providers.md) — write your own policy.
