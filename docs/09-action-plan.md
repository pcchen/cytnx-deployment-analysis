# 09 — Action Plan & Item Tracker

> Upstream tracking issue: Cytnx-dev/Cytnx#970 · derived from [chapter 08](08-upstream-reconciliation.md)

**Living tracker** (last updated 2026-07-05). Consolidates the current status of
every roadmap item, the PR #950 split, and the prioritized next actions. When an
item's status changes, update it here and in `docs/08`.

## Upstream state snapshot (2026-07-05)

- **Issue #970** — "Deployment strategy — status board" (umbrella tracker), OPEN.
- **PR #967** — SDK exclusion (0.1) + OpenBLAS dedup (0.3). Reviewed + **locally
  verified** on manylinux_2_28 aarch64 (single vendored OpenBLAS; SDK excluded);
  follow-up comment posted. Merge-ready pending an x86_64 CI confirmation.
- **PR #950** — CUDA finder hardening + version floors. **Will NOT merge as-is —
  to be split** (see "PR #950 split"). Items 1.1 and 3.2 are `gap` until the
  split PRs land.
- **PR #961** — requires GCC≥13 for `std::format`; **absorb #950's GCC≥13 piece here.**

## Master item status

Classifications: `covered` · `dissolved` · `gap` · `gap-coupled` (blocked on the
#969 packaging-model decision) · `superseded`. ⭐ = passes all three first-PR
gates (real gap · macOS-verifiable · not #969-coupled).

| Item | Status | Upstream | Verifiable on macOS | Next action |
|------|--------|----------|---------------------|-------------|
| **0.1** SDK exclusion from wheel | covered | PR #967 | yes ✅ verified | Land #967; add x86_64 CI check |
| **0.2** Linux slim-LTO `.a` bloat | superseded | (via #967) | n/a | None — disappears with #967 |
| **0.3** OpenBLAS dedup | covered | PR #967 | yes ✅ verified | Land #967; CI single-OpenBLAS assert |
| **1.1** finder `*_FOUND` bug | gap (was in #950) | #945/#946 | no (Linux+CUDA) | **#950 split PR-1** |
| **1.2** `$ORIGIN`/`~` rpath | dissolved | — | — | None |
| **1.3** legacy `CUDA_*_LIBRARY`→`linkflags.tmp` block | gap | #949 | no (Linux+CUDA) | **#950 split PR-3** (extend #949 fix) |
| **1.4** stale `adv_install.rst` options table | gap ⭐ | #950/#967 (adjacent) | yes | Docs PR (bundle B1) |
| **1.5 / #948** CUDA extension not rpath-pinned | gap-coupled | #948 | no (Linux+CUDA) | After #969; needs CUDA CI |
| **2.1** document `pip`/`uv` install | gap ⭐ | — | yes | Docs PR (bundle B1) — first-PR pick |
| **2.2** SciPy S1/S2 pins + documented repair | gap ⭐ | — | yes | Docs + `pyproject.toml` |
| **2.3** conda → conda-forge feedstock + version lag | gap-coupled | — | partial | After #969 |
| **2.4** macOS Accelerate BLAS option | gap ⭐ | — | yes (native) | CMake/recipe option |
| **3.1** CUDA CI runner (build/import `openblas-cuda`) | gap | — | no (needs CUDA runner) | Needs CI hardware |
| **3.2** CuPy-C2 finder search chain | gap (was in #950) | #946 | no (Linux+CUDA) | **#950 split PR-2** |
| **3.3** conda-forge GPU deps (CuPy-C3) | superseded | #969 | — | Folded into #969 |
| **3.4** per-CUDA-major wheels (CuPy-C1) | superseded | #969 | — | A #969 option |
| **3.5** reconcile `cytnx_cuda` conda doc | gap ⭐ | — | yes | Docs PR (bundle B1) |
| **S** packaging model (PyTorch vs JAX) | gap | #969 | n/a | **Drive decision (#969)** |

## PR #950 split

#950 bundles six separable changes; this set covers 100% of it. Detail and file
list in [chapter 08 "PR #950 split"](08-upstream-reconciliation.md#pr-950-split).

| Split PR | Content | Covers | Status |
|----------|---------|--------|--------|
| **PR-1** | finder `*_FOUND` via `find_package_handle_standard_args` | 1.1 | not started — **recommended #950 start** |
| **PR-2** | FindCUTENSOR version-aware `lib/${MAJOR}` subdir | 3.2 | not started (sequence after PR-1) |
| **PR-3** | #949 modern diagnostics **+ remove legacy `linkflags.tmp` block** | #949 + 1.3 | not started |
| **PR-4** | version floors: CUDA≥12 (12&13), cuTENSOR≥2.0, arch adapt, nvcc error | #962 | blocked on #962 decision |
| **(fold)** | GCC≥13 enforcement | — | **fold into PR #961** (no new PR) |
| **PR-6** | `adv_install.rst` dep list + tarball `LD_LIBRARY_PATH` note | #948 doc | not started (bundle B1) |

## Prioritized action items

**A. Upstream coordination (do first — cheap, unblocks the rest)**
- **A1.** Comment on #950 proposing the split plan; get maintainer/author buy-in before opening sub-PRs.
- **A2.** Update status board #970: #967 verified, #950 splitting (1.1/3.2 → gap), link the split plan.

**B. Land the ready PRs**
- **B1.** macOS-verifiable **docs PR bundle** — items 2.1, 1.4, 3.5, and #950 PR-6 (all edit `install.rst`/`adv_install.rst`). One or two focused docs PRs; fully build-verifiable on macOS. **Best starting contribution.**
- **B2.** #950 **PR-1** (finder `*_FOUND` correctness fix) — small, isolated, independent of #962/#961; verify by inspection.

**C. Drive the two blocking decisions**
- **C1. #962** — CUDA version policy (CUDA 12 floor). Unblocks #950 PR-4; clarifies 1.5/2.3.
- **C2. #969** — packaging model (PyTorch- vs JAX-style). Unblocks 1.5, 2.3, 3.3, 3.4, S — the single biggest unblocker.

**D. Deferred (need Linux/CUDA CI or hardware)**
- #950 PR-2, PR-3; item 3.1 (CUDA CI runner); 1.5/#948 (rpath). Park until CUDA CI exists or #969 lands.

**Recommended sequence:** A1 → A2 → B1, then C2 (#969) as the highest-value decision to drive.
