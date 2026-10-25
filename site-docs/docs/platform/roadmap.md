# Track graduation roadmap

The platform tracks below ship today as **incubating `0.x` modules**. This page
documents the plan to promote each of them to **stable `1.0`** — track by track,
in dependency order, without breaking the Runtime 1.0-RC1 contract or the
[ADR-016](https://github.com/okostiukevych/CortexJ/blob/main/docs/adr/ADR-016-track-boundaries.md)
dependency boundaries. The umbrella tracking issue for the whole campaign is
[#189](https://github.com/okostiukevych/CortexJ/issues/189).

!!! warning "Sequencing, not scheduling"
    The phases below are ordered by dependency, not by calendar. No dates are
    promised: a track graduates only when every gate is closed and CI-verified,
    and the maturity label changes only at the moment the `<track>-1.0.0` tag is
    published.

## Status today

| Track | Maturity | Graduation scope |
|---|---|---|
| Runtime / inference | beta (1.0.1) | not in scope — already the production product |
| Data | incubating | **pilot promotion** — smallest closed contract |
| Experiments | incubating | early promotion, after Data process is proven |
| Lab / Notebook | incubating | renderers, protocol conformance, cleanup, OS matrix |
| Tensor / Autograd | incubating | the largest gate: op conformance, gradient suite, serialization |
| Train / PEFT | incubating | checkpoint/resume, precision semantics, provenance capture |
| Research | incubating | first: publish its Definition of Done (a documentation gap today) |

See the [platform overview](index.md) for what each track contains today and the
[API stability tiers](../reference/api-stability.md) for what the 0.x label
means contractually.

## What "graduation" means — the promotion gate

Removing the incubating label is a statement about the **contract**, not about
feature parity. Every promotion issue must close all eight gates of the campaign
contract:

1. No `experimental.*` packages remain in the track's public contracts
   (promoted or moved; recorded in the [API stability](../reference/api-stability.md) doc).
2. A TCK contract exists for every public SPI of the track and runs against all
   registered providers in CI.
3. The track publishes its own semver `1.0.0` (it stops inheriting the reactor
   root version).
4. Binary compatibility is enforced per track (japicmp baseline): a
   breaking change fails CI without a major bump.
5. A smoke/demo is green in CI for the track.
6. All documentation and site labels are updated (track page, maturity table,
   API stability, README, landing page).
7. A promotion ADR is published: what is frozen, what remains follow-up — with
   issue numbers.
8. A scope-gate report confirms no claim exceeds what tests prove.

## Sequencing

| Phase | Work | Key gate closed | Issues |
|---|---|---|---|
| G0 — infrastructure | per-track versions, TCK runner for track SPIs, per-track japicmp profile, graduation runbook, Research Definition of Done written down | graduation becomes mechanically possible for any track | [#190](https://github.com/okostiukevych/CortexJ/issues/190)–[#194](https://github.com/okostiukevych/CortexJ/issues/194) |
| Data (pilot) | all 8 adapters through `DataSourceContract`; cross-adapter fingerprint determinism | first promotion — the process itself is validated | [#195](https://github.com/okostiukevych/CortexJ/issues/195)–[#197](https://github.com/okostiukevych/CortexJ/issues/197) |
| Experiments | store TCK, provenance schema versioning, retention policy | second promotion | [#198](https://github.com/okostiukevych/CortexJ/issues/198)–[#201](https://github.com/okostiukevych/CortexJ/issues/201) |
| Lab / Notebook | chart/image renderers, cancellation & cleanup tests, Jupyter wire-protocol conformance, env manifest + export smoke, 3-OS CI matrix | maturity gate point-by-point | [#202](https://github.com/okostiukevych/CortexJ/issues/202)–[#207](https://github.com/okostiukevych/CortexJ/issues/207) |
| Tensor / Autograd | golden fixtures ≥25 ops, CPU + Vector backend conformance, shape/broadcast property tests, gradient suite, memory lifecycle, serialization, MLP demo | maturity gate point-by-point — the largest gate | [#208](https://github.com/okostiukevych/CortexJ/issues/208)–[#217](https://github.com/okostiukevych/CortexJ/issues/217) |
| Train / PEFT | backend TCK (DJL + remote), E2E checkpoint/resume determinism, memory planner fail-early, explicit mixed-precision semantics, full provenance capture, fail-fast combos | maturity gate point-by-point | [#218](https://github.com/okostiukevych/CortexJ/issues/218)–[#224](https://github.com/okostiukevych/CortexJ/issues/224) |
| Research | card schema + store SPI + TCK, citations & rendering, benchmark-harness demo | ratified DoD of its own | [#225](https://github.com/okostiukevych/CortexJ/issues/225)–[#227](https://github.com/okostiukevych/CortexJ/issues/227) |

Research graduates strictly after Tensor/Autograd: the `nn` modules depend on
the stabilized Tensor/Autograd contract.

## Per-track work in the campaign

- **Data** — prove content-derived fingerprinting across adapters (CSV ↔ JSON ↔
  Parquet give identical fingerprints), strict null/empty semantics (no silent
  coercion to `0`/`""`), and hold every adapter to the same behavioural contract.
- **Experiments** — reference contract for the experiment store (any third-party
  tracker must pass it; external trackers like MLflow stay a follow-up, not a 1.0
  blocker), versioned provenance schema with backward-compat guarantees, and an
  explicit retention default.
- **Lab** — honest closure of the maturity gate: charts/images render in notebooks, cancellation
  and resource cleanup is test-verified, the Jupyter wire protocol has a
  conformance suite, exported notebook → Maven projects build green in CI, and
  the kernel is verified on ubuntu/macos/windows.
- **Tensor / Autograd** — the CPU reference backend moves from zero tests to a
  full conformance suite with golden fixtures from a trusted reference; the same
  suite then gates the Vector backend; memory ownership, serialization format
  and broadcast semantics become documented contracts.
- **Train** — a real fine-tune job (LoRA, synthetic data, CPU) proves
  train → checkpoint → resume with documented numerical tolerance; unsupported
  backend/precision combos fail before expensive execution; every run captures a
  complete provenance manifest (dataset hash, model revision, hyperparameters,
  seed).
- **Research** — first, a proper Definition of Done is written down
  (today's known gap); then cards, citations and the benchmark harness are
  stabilized on contract terms.

## How progress is tracked

- The umbrella issue
  [#189](https://github.com/okostiukevych/CortexJ/issues/189) carries the
  gate-checklist of record; every promotion issue is a child of that checklist.
- AFK-labelled issues can be picked up and merged without human interaction; the
  `hitl`-labelled issues (six promotions plus two infrastructure decisions)
  require maintainer sign-off.
- This page is updated as phases land; the landing-page maturity labels change
  only at release time.

!!! info "Honesty policy"
    Graduation claims may not exceed what tests prove. Anything closed partially
    stays listed as follow-up in the promotion ADR and never enters the 1.0
    contract silently.
