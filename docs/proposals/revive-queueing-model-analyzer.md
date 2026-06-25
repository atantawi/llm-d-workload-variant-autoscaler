# Proposal: Revive the QueueingModelAnalyzer (SLO-driven, model-based scaling)

**Authors:** Asser Tantawi
**Status:** Draft
**Created:** 2026-06-25
**Last Updated:** 2026-06-25

---

## Summary

The **QueueingModelAnalyzer (QMA)** is an SLO-driven, model-based capacity analyzer that already
lives in the WVA tree (`internal/engines/analyzers/queueingmodel/`). It uses queueing theory and an
online Extended Kalman Filter (EKF) to learn per-hardware service parameters and size replicas so that
explicit per-model latency SLOs (TTFT, ITL) are met — with no manual calibration.

QMA was merged in March 2026 (PR #791, "SLO-driven model-based scaling") as an experimental feature,
gated behind a dedicated ConfigMap, and **was never validated end-to-end**. Since then WVA's analyzer
architecture has evolved into a **multi-analyzer pipeline** (June 2026), and the Throughput analyzer
was wired into it — but QMA was left behind on a legacy code path. In parallel, the research prototype
it descends from (llm-inferno) has gained substantial robustness improvements that QMA does not have.

This proposal makes the case for **reviving QMA** in two phases: (1) re-home it as a first-class peer in
the multi-analyzer pipeline and validate it end-to-end; (2) selectively adopt proven upstream
robustness work. It is intended to open a team discussion, not to prescribe a final implementation.

---

## Problem Statement

QMA is dormant, unvalidated, and architecturally stranded:

- **It bypasses the multi-analyzer pipeline.** WVA today has two *mutually exclusive* analyzer
  architectures. QMA sits on the older **"select-one" path**
  (`internal/engines/saturation/engine.go:441`, `switch analyzerName`), and is activated simply by the
  presence of the `wva-queueing-model-config` ConfigMap, which **overrides everything else**
  (`engine.go:413`) and routes to a bespoke `e.optimizeQueueingModel()` path
  (`internal/engines/saturation/engine_queueing_model.go`). Turning QMA on therefore *disables* the
  saturation and throughput analyzers entirely — it cannot run alongside them.
- **It was never tested end-to-end.** Unit tests exist for the parameter store, utility conversions,
  and the tuner, but there is **no integration or e2e coverage** of the full
  metrics → tune → SLO → capacity → decision pipeline. The feature has never been exercised on a
  live cluster.
- **Its math has diverged from upstream.** The queueing model (`pkg/analyzer/queueanalyzer.go`) and the
  tuner (`internal/engines/analyzers/queueingmodel/tuner/`) are **independent reimplementations** of
  the llm-inferno `queue-analysis` and `model-tuner` modules. Only `kalman-filter` is actually imported
  (`go.mod`). Upstream has since fixed real-world failure modes (degenerate fits, EKF divergence,
  cold-start over-estimation) that QMA remains exposed to.
- **It carries known gaps and TODOs**, e.g. SLO smoothing (`analyzer.go:116`), an unimplemented
  covariance validation step (`tuner/tuner.go:285`), hardcoded batch/queue sizes (`defaults.go:5`),
  an undecided pod-aggregation strategy (`analyzer.go:824-826`), and a hardcoded
  `TuningByAggregatingPodsForVariant` toggle.

The cost of leaving QMA as-is is real: it is dead weight in the codebase, it deprives WVA of an
SLO-aware scaling option, and the longer it drifts from upstream the more expensive any future revival
becomes.

---

## Motivation / Impact (the "why")

WVA's current analyzers answer "is the fleet saturated?" (saturation, reactive, KV-token based) and
"is token throughput about to be outrun?" (throughput, proactive, token-rate based). **Neither sizes
capacity to an explicit latency SLO.** QMA fills exactly that gap:

- **Latency SLOs as a first-class input.** Operators can state per-model TTFT/ITL budgets and have the
  autoscaler size replicas to honor them, rather than tuning saturation/throughput thresholds as
  latency proxies.
- **Self-calibrating.** The EKF learns the hardware service parameters `(alpha, beta, gamma)` online
  from observed TTFT/ITL — no offline benchmarking or per-accelerator constants to maintain.
- **Complementary in the pipeline.** The pipeline combines analyzers as *scale-up = ANY*,
  *scale-down = ALL*. As a peer analyzer, QMA can (a) **add SLO-protective scale-ups** the other
  analyzers would miss, and (b) act as an **SLO veto** that prevents a scale-down which would breach
  latency budgets. It strengthens the existing analyzers rather than replacing them.

The impact is a production-grade, SLO-driven scaling option that no other analyzer currently provides —
turning an abandoned experiment into a differentiated capability.

---

## Current State

| Component | Path |
|-----------|------|
| QueueingModelAnalyzer | `internal/engines/analyzers/queueingmodel/analyzer.go` |
| Config / parameter store / defaults | `internal/engines/analyzers/queueingmodel/{config,parameters,defaults}.go` |
| Tuner (EKF wrapper) | `internal/engines/analyzers/queueingmodel/tuner/` |
| Queue model | `pkg/analyzer/queueanalyzer.go`, `pkg/analyzer/queuemodel.go` |
| Legacy activation path | `internal/engines/saturation/engine_queueing_model.go`, `engine.go:413,441` |
| ConfigMap interface | `internal/interfaces/queueing_model_scaling.go` (`QueueingModelAnalyzerName = "queueing-model"`) |
| ConfigMap YAML | `deploy/configmap-queueing-model.yaml` |
| Developer guide | `docs/developer-guide/slo-queuemodel.md` |

- **Implements `interfaces.Analyzer` already** — `Name()` returns `"queueing-model"` and `Analyze()`
  returns an `*AnalyzerResult`, the same contract the pipeline uses. The hard part of re-homing is
  mostly *wiring and retirement of the legacy path*, not rewriting the analyzer.
- **Tested:** parameter store (`parameters_test.go`), utils (`utils_test.go`), config validation
  (`queueing_model_scaling_test.go`), and the tuner. **Untested:** the end-to-end analyzer pipeline.
- **Known TODOs / toggles:** see Problem Statement.

The full algorithm and theory are documented in `docs/developer-guide/slo-queuemodel.md` (this proposal
does not restate the math).

---

## Relationship to the Multi-Analyzer Pipeline

The throughput analyzer is the template for how a model-based analyzer should plug into WVA today:

- It implements `interfaces.Analyzer`, is registered via `engine.RegisterAnalyzer(...)`
  (`cmd/main.go:486`), and is gated by an `analyzers:` entry in the saturation-scaling config (an
  `enabled` flag checked at startup, `cmd/main.go:106`).
- The engine runs saturation-V2 first (for per-variant metadata), then each registered analyzer,
  applies the universal threshold post-step to each result, and the optimizer combines them
  (scale-up = ANY, scale-down = ALL). See `docs/developer-guide/multi-analyzer-pipeline.md`.

**The recommended path for QMA is to become exactly such a peer:** register it like throughput, gate it
with an `analyzers:` config entry, and **retire** the ConfigMap-override activation and the
`optimizeQueueingModel` / `engine_queueing_model.go` legacy path. This lets QMA coexist with saturation
and throughput, inherits the universal threshold + combine semantics for free, and removes a
special-case branch from the engine.

> The QMA-specific `wva-queueing-model-config` ConfigMap (per-model SLO targets, `sloMultiplier`,
> `tuningEnabled`) remains useful as QMA's *own* configuration surface; only its role as the
> pipeline-bypassing activation switch is retired.

---

## Relationship to llm-inferno

QMA mirrors three llm-inferno modules, but the projects have different missions:

| llm-inferno (research prototype) | WVA (production) |
|----------------------------------|------------------|
| `control-loop` (controller + collector + actuator + optimizer) | WVA engine + reconcile loop |
| `queue-analysis` | `pkg/analyzer/queueanalyzer.go` |
| `model-tuner` (EKF + Nelder-Mead estimators) | `internal/engines/analyzers/queueingmodel/tuner/` |

**Explicitly out of scope** for WVA: the collector and actuator (WVA has its own metrics collection and
KEDA/HPA actuation), and the optimizer's extra decision variables. llm-inferno optimizes
`{GPU type, max batch size, numReplicas}`; **WVA's only decision variable is `numReplicas`** — capacity
sizing here is purely "allocate available replicas to meet demand," so no MILP/GPU-type optimizer is
needed.

WVA does not aim to track llm-inferno feature-for-feature. The two overlap but are not competitive:
WVA selectively adopts the *algorithms that have proven robust* in the prototype, on WVA's own terms
and interfaces.

---

## Dependency Strategy (recommendation)

QMA's queue model and tuner are currently local reimplementations. We **recommend importing the
upstream llm-inferno modules via `go.mod`** — `github.com/llm-inferno/queue-analysis` and
`github.com/llm-inferno/model-tuner` — consistent with how WVA already depends on
`github.com/llm-inferno/kalman-filter v0.1.2`.

**Why import:**
- Inherit upstream bug fixes and robustness work (below) without re-deriving them.
- Reduce drift: the math stays in one authoritative place.
- Lower long-term maintenance — WVA wraps stable, versioned modules behind its own interfaces.

**Alternative — keep a local fork:**
- *Pro:* full control, no external coupling, can diverge freely for production needs.
- *Con:* every upstream fix must be manually re-ported (the situation that produced today's drift);
  higher maintenance cost.

Either way, an **interface-adaptation layer** is required so WVA's `interfaces.Analyzer`,
`AnalyzerInput`, and `AnalyzerResult` types map onto the upstream module APIs. This decision (import
vs. fork, and at which module versions) should be settled early in Phase 2 as it shapes the port.

---

## Recent llm-inferno features to adopt

These upstream improvements (Feb–Apr 2026) target failure modes QMA is currently exposed to. Several
map directly onto QMA's existing TODOs.

| Upstream feature (module) | WVA gap / TODO it addresses | Benefit |
|---------------------------|-----------------------------|---------|
| **Identifiability guard** — condition-number check rejects degenerate fits (`model-tuner`, Apr 2026) | No covariance/quality validation beyond NIS (`tuner/tuner.go:285`) | Prevents `(alpha,beta,gamma)` collapse at single-replica / low-utilization — WVA's common case |
| **Sliding-Window Nelder-Mead estimator + EKF fallback** (`model-tuner`, Mar–Apr 2026) | EKF is the only estimator; no fallback when it diverges | Robust parameter learning when the EKF is unstable |
| **Multi-observation init + warm-start** (`model-tuner`, Mar 2026) | Cold-start bootstrap can over-estimate `alpha`; single-observation seeding | Better cold-start accuracy, faster convergence, fewer early mis-scales |
| **Optimal-concurrency search** (`queue-analysis` `/optimize`, Feb–Mar 2026) | Hardcoded `DefaultMaxBatchSize`/`DefaultMaxQueueSize` (`defaults.go:5`) | Formula-guided batch sizing instead of hardcoded defaults (optional for WVA) |
| **Overload handling / `OfferedRate`** (`queue-analysis`, Feb 2026) | No explicit offered-vs-served distinction under overload | Correct behavior when demand exceeds a replica's max sustainable rate |
| **SLO time-series smoothing** | `analyzer.go:116` (inferred SLO fluctuates per cycle) | Stable SLO targets, less scaling jitter |

---

## Phased Plan

### Phase 1 — Re-home & validate (Alpha)
- Register QMA as a pipeline analyzer (peer of saturation/throughput) via `RegisterAnalyzer`, mirroring
  the throughput wiring in `cmd/main.go`.
- Add an `analyzers:` config entry with an `enabled` gate for `"queueing-model"`.
- Retire the ConfigMap-override activation and the `optimizeQueueingModel` / `engine_queueing_model.go`
  legacy branch; keep `wva-queueing-model-config` as QMA's own config surface.
- Add integration + e2e tests covering metrics → tune → SLO resolution → capacity → decision, and
  verify QMA coexists correctly with saturation/throughput under the combine rules.
- Update `docs/developer-guide/slo-queuemodel.md` to reflect pipeline-based activation.

### Phase 2 — Adopt upstream robustness (Beta+)
- Settle the dependency decision (import vs. fork; module versions).
- Port the identifiability guard, SWNM + EKF-fallback, and multi-observation init/warm-start.
- Optionally adopt optimal-concurrency search and overload handling.
- Implement SLO time-series smoothing and resolve the pod-aggregation strategy
  (`analyzer.go:824-826`).

---

## Open Questions / Risks

- **Cold-start under-provisioning** vs. over-provisioning during EKF warm-up (currently 1.5× observed
  latency fallback).
- **Single-replica identifiability** — limited token-count spread makes `(alpha,beta,gamma)` hard to
  separate; the upstream guard mitigates but does not eliminate this.
- **Per-pod vs. aggregated tuning** — `TuningByAggregatingPodsForVariant` is hardcoded; which strategy
  is correct for WVA?
- **HA warm-up parity** — like the throughput analyzer, QMA's learned state is in-memory; behavior on
  leader failover needs definition (saturation covers during warm-up).
- **Dependency coupling** — importing upstream modules ties WVA release cadence to llm-inferno tags.
- **Combine-rule interactions** — validate that QMA's SLO veto on scale-down does not deadlock against
  cost-driven scale-downs from other analyzers.

---

## References

- `docs/developer-guide/slo-queuemodel.md` — QMA algorithm and theory
- `docs/developer-guide/multi-analyzer-pipeline.md` — pipeline architecture and combine rules
- `docs/developer-guide/throughput-analyzer.md` — the reference peer analyzer
- llm-inferno modules: `github.com/llm-inferno/{queue-analysis,model-tuner,kalman-filter}`
- PR #791 (QMA original); PRs #1228 / #1246 / #1250 / #1266 (multi-analyzer pipeline + throughput)
