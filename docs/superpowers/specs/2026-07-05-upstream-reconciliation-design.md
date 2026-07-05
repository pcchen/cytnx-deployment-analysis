# Cytnx Deployment Roadmap — Upstream Reconciliation & Status Board — Design Spec

**Date:** 2026-07-05
**Target repo:** `github.com/Cytnx-dev/Cytnx` (default branch `master`; author has `admin`/`push`)
**Planning home:** this analysis repo (`cytnx_design_deployment`)
**Status:** Approved design, ready for implementation planning

## Purpose

Turn the ch.06 recommended strategy (and ch.07 wheel-efficiency findings) into
upstream action. The central discovery that shapes this project: **much of the
roadmap is already in flight upstream** — and part of it is **superseded** by a
newer upstream study. Before proposing or implementing anything, we must
reconcile our roadmap against what already exists, then give the upstream
cluster the phased structure it currently lacks.

This is the **first sub-project** of a larger campaign to implement ch.06 as a
series of upstream PRs. Later phases (Phase 2 packaging, Phase 3 GPU) each get
their own spec → plan → implementation cycle. This spec deliberately produces
**no code PR** — it produces the map that tells us which PR to write first.

### Why reconciliation first (the discovered upstream state)

As of 2026-07-05 the upstream repo already contains, much of it authored by
Claude Code in prior sessions from this very analysis:

- **Strategic (Phases 2–3):** #969 *"Investigate CPU/CUDA wheel packaging
  strategy for PyPI, modeled on PyTorch and JAX"* — a primary-source study that
  reframes our ch.04→06 packaging recommendation as an explicit, still-**undecided**
  model gate (PyTorch-style single package vs JAX-style separate
  `cpu`/`cuda12`/`cuda13` variants). #966 (shared `libcytnx.so`) is adjacent.
- **Tactical build (Phase 1):** PR #950 hardens the cuTENSOR/cuQuantum finders
  and fixes stale CUDA diagnostics (covers #945/#946 and the configure half of
  #949); issue #948 (installed CUDA extension not rpath-pinned to the build
  toolkit) is an **open gap** #950 explicitly defers.
- **Packaging (Phase 0):** PR #967 excludes the C++ SDK from the Python wheel
  (addresses #947); the **OpenBLAS-dedup** half of #947 is still an open gap.

Consequences already established during brainstorming:

- Roadmap item **1.2** ("add `$ORIGIN`, remove the unexpanded `~`") is
  **dissolved** — the current source block uses `@loader_path` /
  `${CMAKE_INSTALL_PREFIX}/lib` with no `~`; the v1.1.0 finding is stale.
- The two biggest genuine gaps are each **complicated**: #948 (CUDA rpath) is
  both un-verifiable on macOS and coupled to the undecided #969 model; OpenBLAS
  dedup is a manylinux-only concern. So the first code PR must be *chosen*, not
  assumed.

## Deliverables

Exactly two artifacts, plus one named output. **No code PR is written in this
sub-project.**

1. **Reconciliation map** — `docs/08-upstream-reconciliation.md` in this repo.
2. **Umbrella tracking issue** — one issue on `Cytnx-dev/Cytnx`.
3. **First-PR candidate** — named and justified inside the map (Section, below),
   becoming the input to the *next* spec.

### Deliverable 1 — Reconciliation map (`docs/08-upstream-reconciliation.md`)

One row per ch.06 roadmap step (Phases 0–3) **and** each ch.07 wheel item. Columns:

| Column | Meaning |
|--------|---------|
| Item | The roadmap step (short label) |
| Our ref | Where we said it (e.g. `ch.06 §6.3 Phase 1`, `ch.07 §7.3`) |
| Upstream issue/PR | Linked `Cytnx-dev/Cytnx` issue/PR number(s), or `—` |
| Status | `merged` / `open-PR` / `open-issue` / `none` |
| Verified on `master`? | Whether the cited code state was confirmed against a `master` checkout (v1.1.0 line numbers are stale and must be re-checked) |
| Classification | one of: `covered` · `dissolved` · `gap` · `gap-decision-coupled` · `superseded` |

**Classifications defined:**

- `covered` — an open/merged PR already does this (e.g. 0.1/#967, 1.1/#950).
- `dissolved` — the v1.1.0 finding no longer holds on `master` (e.g. 1.2's `~`).
- `gap` — real, unaddressed, and actionable now.
- `gap-decision-coupled` — real gap, but its correct fix depends on the #969
  packaging-model decision (e.g. #948's rpath target).
- `superseded` — our recommendation is subsumed/reframed by a stronger upstream
  artifact, chiefly #969 for Phases 2–3.

**Method (per row):** `gh issue view` / `gh pr view` for the upstream status,
then `git -C <master-clone>` diff/grep of the cited file to confirm the current
code state. Every "Verified on master?" cell must be filled from the clone, not
from the v1.1.0 submodule.

The map ends with a short **"First-PR candidate"** section applying the three
gates below.

### Deliverable 2 — Umbrella tracking issue (upstream)

A single issue, "Deployment strategy — status board", that **references, never
duplicates** the existing cluster. Structure:

- **Phase gate (Phases 2–3):** #969 (+#966) — "decide the packaging model."
- **Phase 1 (build discovery):** #945/#946/#949 → PR #950; #948 as the open gap.
- **Phase 0 (quota/wheels):** #947 → PR #967; OpenBLAS-dedup as the open gap.
- Each line a checkbox linking its issue/PR; genuine gaps flagged; a one-line
  note wherever our ch.06/07 is `superseded` by #969.

Its job is to give the scattered #945–#969 cluster the single phased view it
lacks — not to restate the roadmap.

### Deliverable 3 — First-PR candidate (an output, not code)

The map names the next code PR by three gates, **all required**:

1. **Genuine gap** — classification `gap` (not `covered`/`dissolved`/`superseded`).
2. **macOS-verifiable** — buildable/observable on the author's Mac (natively, or
   via cibuildwheel-in-Docker for manylinux); no Linux+CUDA-only repro.
3. **Not decision-coupled** — not `gap-decision-coupled` on #969.

#948 fails gates 2 and 3 and is therefore explicitly **not** the first PR.
OpenBLAS-dedup must have its gate-2 (Docker) verifiability confirmed in the map.

## Environment & workflow

- **Master clone:** a fresh clone of `Cytnx-dev/Cytnx` at `master` in a sibling
  directory (`../Cytnx-upstream/`), separate from this repo's read-only
  `external/Cytnx` v1.1.0 submodule. This repo stays the planning/analysis home;
  the clone is only for verifying current code state (and, in later specs, for
  PR branches).
- **Platform ceiling (macOS/darwin):** recorded per row in the map. Linux+CUDA
  items (e.g. #948) cannot be built or verified locally; manylinux-only items
  may be verifiable via cibuildwheel Docker; macOS-native items verify natively.
- **No fork needed:** author has `admin`/`push` on upstream; the umbrella issue
  is filed directly. `gh` token carries `repo` + `workflow` scopes.

## Out of scope

- Any **code PR** (identified here, implemented in the next spec).
- Executing **Phases 2–3**, and making the **#969 packaging-model decision**.
- Re-litigating `covered` items (0.1/#967, 1.1/#950, 1.3).

## Success criteria

1. `docs/08-upstream-reconciliation.md` committed: every ch.06 (Phases 0–3) and
   ch.07 item present, classified, with its "Verified on `master`?" cell filled
   from the clone.
2. The umbrella tracking issue is live on `Cytnx-dev/Cytnx`, links the full
   #945–#969 cluster under the phased structure, and duplicates none of it.
3. The map names exactly one **first-PR candidate** that passes all three gates,
   with its verifiability verdict stated — ready to seed the next spec.
