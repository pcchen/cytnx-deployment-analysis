# cytnx-deployment-analysis

Analysis of how [Cytnx](https://github.com/Cytnx-dev/Cytnx) (>= 1.1.0) is
deployed today, and a recommended deployment strategy for the maintainers.

The Cytnx source is pinned as a reference-only submodule under `external/Cytnx`
(v1.1.0). This repo proposes; it does not modify Cytnx.

## TL;DR

Ship a two-channel deployment strategy split by dependency tier: **pip/wheels (with uv)
as the primary channel for the already-working CPU story, and the conda-forge ecosystem
(with pixi) as the primary channel for the GPU stack and runtime robustness**, with
build-from-source demoted to a fallback. Full rationale and the highest-impact fixes are
in the [executive summary](docs/00-executive-summary.md).

## How to read this

1. [Executive summary](docs/00-executive-summary.md)
2. [Current-state audit](docs/01-current-state-audit.md)
3. [Dependency taxonomy](docs/02-dependency-taxonomy.md)
4. [Core problems](docs/03-core-problems.md)
5. [Reference projects](docs/04-reference-projects.md)
6. [Channel evaluation](docs/05-channel-evaluation.md)
7. [Recommended strategy & roadmap](docs/06-recommended-strategy.md)
8. [Wheel-artifact efficiency](docs/07-wheel-artifact-efficiency.md)
9. [Upstream reconciliation & status board](docs/08-upstream-reconciliation.md)
10. [Action plan & item tracker](docs/09-action-plan.md)

## Design & process

How this analysis was scoped and built:

- [Design spec](docs/superpowers/specs/2026-07-03-cytnx-deployment-analysis-design.md) — the agreed scope, structure, and success criteria for the document.
- [Implementation plan](docs/superpowers/plans/2026-07-03-cytnx-deployment-analysis.md) — the task-by-task plan used to write and verify each chapter.

## Status

Analysis complete. Implementation campaign underway: the ch.06/07 roadmap is
reconciled against upstream in [chapter 08](docs/08-upstream-reconciliation.md),
and tracked upstream at Cytnx-dev/Cytnx#970. The first gap PR is
named in chapter 08's "First-PR candidate" section; each phase is a separate
spec → plan → PR cycle.
