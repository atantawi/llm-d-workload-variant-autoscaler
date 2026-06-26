## Motivation

WVA can scale on saturation (reactive, KV-token based) and throughput (proactive, token-rate based),
but **neither sizes capacity to an explicit latency SLO**. The `QueueingModelAnalyzer` (QMA) — already
in the tree under `internal/engines/analyzers/queueingmodel/` — does exactly that: operators state
per-model TTFT/ITL budgets, and the analyzer learns the hardware's service parameters online (via an
Extended Kalman Filter) and sizes replicas to meet those budgets, with no manual calibration.

QMA was merged in March 2026 (PR #791) as experimental, gated behind a dedicated ConfigMap, and **never
validated end-to-end**. Since then WVA grew a **multi-analyzer pipeline** (June 2026) and wired the
Throughput analyzer into it — but QMA was left on a legacy "select-one" code path that *bypasses* the
pipeline (turning QMA on today disables saturation + throughput). Meanwhile its upstream research
prototype, llm-inferno, has added robustness fixes (identifiability guards, EKF-fallback estimator,
better cold-start) that QMA's local reimplementation does not have.

Reviving QMA turns an abandoned experiment into a differentiated, SLO-driven scaling capability that
**complements** the existing analyzers: under the pipeline's combine rules it can add SLO-protective
scale-ups and veto SLO-breaching scale-downs. The longer it drifts, the more expensive revival becomes.

### Enhancement Description

- One-line enhancement description (can be used as a release note): Revive the QueueingModelAnalyzer as
  an SLO-driven analyzer in the multi-analyzer pipeline, and align it with proven llm-inferno robustness
  work.
- Enhancement Proposal: [`docs/proposals/revive-queueing-model-analyzer.md`](./revive-queueing-model-analyzer.md)
- PRs by stage and milestone:
  - [ ] Alpha - v0.xx — **Phase 1: re-home & validate** (register QMA as a pipeline peer like the
    throughput analyzer; add an `analyzers:` config gate; retire the ConfigMap-override /
    `optimizeQueueingModel` legacy path; add integration + e2e tests; update the developer guide)
    - [ ] Code (`llm-d/llm-d-workload-variant-autoscaler`) update PR(s):
    - [ ] Docs (`llm-d/llm-d`) update PR(s):
    - [ ] Guides (`llm-d/llm-d`) update PR(s):

<!-- Uncomment when using AI assistants to generate designs and plans
- Design Document (the how): docs/specs/...
- Plan Document (the when and where): docs/plans/...
-->

<!-- Uncomment these as you prepare the enhancement for the next stage
- [ ] Beta - v0.xx — Phase 2: adopt upstream llm-inferno robustness (dependency decision: import via
      go.mod vs. local fork; identifiability guard, sliding-window Nelder-Mead + EKF fallback,
      multi-observation init/warm-start; optional optimal-concurrency & overload handling; SLO smoothing)
  - [ ] Code (`llm-d/llm-d-workload-variant-autoscaler`) update PR(s):
  - [ ] Docs (`llm-d/llm-d`) update PR(s):
  - [ ] Guides (`llm-d/llm-d`) update PR(s):
- [ ] Stable - v0.xx
  - [ ] Code (`llm-d/llm-d-workload-variant-autoscaler`) update PR(s):
  - [ ] Docs (`llm-d/llm-d`) update PR(s):
  - [ ] Guides (`llm-d/llm-d`) update PR(s):
-->

_Please keep this description up to date. This will help the Enhancement Team to track the evolution of the enhancement efficiently._
