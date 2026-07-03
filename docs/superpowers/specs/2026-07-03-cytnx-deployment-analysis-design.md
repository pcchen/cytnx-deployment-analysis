# Cytnx Deployment Analysis — Design Spec

**Date:** 2026-07-03
**Repo (to be created):** `github.com/pcchen/cytnx-deployment-analysis`
**Status:** Approved design, ready for implementation planning

## Purpose

Produce a written **analysis + strategy design document** that examines how
Cytnx (≥ 1.1.0) is currently deployed and recommends a better deployment
strategy. The document is authored **for the Cytnx maintainers (Cytnx-dev)** and
is intended to be actionable — a basis for concrete upstream changes, issues, or
a roadmap/PR.

This project delivers the **analysis and recommendation only**. Building working
deployment configurations (pixi/conda/pip/uv prototypes) is a **clearly-marked
follow-up phase**, described in the roadmap but not implemented here.

## Deliverable

A GitHub repository that is, in effect, an authoritative research report with the
Cytnx source pinned as reference material.

### Repository structure

```
cytnx-deployment-analysis/                 (github.com/pcchen/)
├── README.md                  # what this repo is, how to read it, TL;DR of the recommendation
├── external/
│   └── Cytnx/                 # git submodule, pinned at the v1.1.0 tag (reference-only)
├── docs/
│   ├── 00-executive-summary.md
│   ├── 01-current-state-audit.md
│   ├── 02-dependency-taxonomy.md
│   ├── 03-core-problems.md            # build-time discovery + import-time resolution
│   ├── 04-reference-projects.md       # SciPy, CuPy
│   ├── 05-channel-evaluation.md       # source / pip / conda / pixi / uv
│   └── 06-recommended-strategy.md     # + roadmap for maintainers
├── assets/                    # diagrams (dependency graph, load-time resolution flow)
└── LICENSE
```

- The Cytnx submodule is **reference-only**: we read it and cite specific files
  (and line/section) but never modify it. Pinning to the **v1.1.0 tag** makes
  every citation stable and reproducible. (Original brief said "fix the comment"
  for the submodule; interpreted as **pin the commit**.)
- Each `docs/NN-*.md` chapter is focused and independently readable.
- `00-executive-summary.md` is written last but read first.

## Document structure (layered / problem-first)

Chosen over dependency-centric or channel-centric organization because it is the
only structure that both guarantees per-dependency coverage **and** drives a
single clear decision for the maintainer audience.

### 00 — Executive summary
One page: the recommendation, the 2–3 highest-impact changes for maintainers,
and a pointer to the follow-up prototype phase. Written last.

### 01 — Current-state audit
Factual reconstruction of how Cytnx v1.1.0 builds and ships **today**, cited to
submodule files:
- Build system: `CMakeLists.txt`, `CytnxBKNDCMakeLists.cmake`,
  `CMakePresets.json`, `version.cmake`, and the `USE_*` option matrix
  (`USE_CUDA`, `USE_MKL`, `USE_HPTT`, `USE_CUTENSOR`, `USE_CUQUANTUM`,
  `BUILD_PYTHON`, `BACKEND_TORCH`).
- Distribution today: `pyproject.toml`, the `conda_build/` recipe, and
  `.github/workflows/` — what wheels/packages are actually produced, for which
  platforms / Python versions / CUDA versions.
- Install paths as documented for users (pip / conda / source-only).

### 02 — Dependency taxonomy
The required native libraries classified by **shipping difficulty**:
- OpenBLAS / MKL, hptt, CUDA toolkit, cuTENSOR, cuQuantum, ARPACK, pybind11,
  Boost. (Build-system evidence: ARPACK via `find_library(ARPACK_LIB arpack
  REQUIRED)`; Boost via `find_package(Boost REQUIRED)`; pybind11 via FetchContent
  when not found locally.)
- Per dependency: role, how Cytnx currently finds it (build-time) and loads it
  (runtime), ABI/versioning constraints, redistribution/licensing constraints,
  and whether a PyPI wheel / conda-forge package already exists.
- Note the distinct shipping profiles: pybind11 is header-only (build-time only,
  no runtime lib); Boost's footprint depends on which components are used
  (header-only vs. compiled); ARPACK is a compiled runtime dependency that must
  be found and loaded like BLAS.

### 03 — Core problems (the one story)
Build-time discovery and import-time runtime resolution analyzed **together** as
a single narrative:
- **Build-time:** `find_package` vs. environment-variable discovery
  (`CUTENSOR_ROOT` / `CUQUANTUM_ROOT`) inconsistency; reproducibility gaps.
- **Import-time:** how the installed extension resolves native libraries at
  `import cytnx` (`RPATH` / `LD_LIBRARY_PATH`, bundled vs. system libs); failure
  modes (wrong library found, ABI mismatch, GPU driver/toolkit skew); and the
  pip-vs-conda library-conflict story.

### 04 — Reference projects
How **SciPy** and **CuPy** solve exactly these problems, grounded in their public
build/packaging files and docs, distilled into named, transferable patterns:
- **SciPy** — the BLAS / manylinux / wheel-bundling playbook.
- **CuPy** — the CUDA-runtime / cuTENSOR / cuQuantum + per-CUDA-version wheel
  playbook.

### 05 — Channel evaluation
A matrix scoring **build-from-source, pip/wheels, conda(-forge), pixi, and uv**
against: dependency coverage, GPU support, runtime-resolution robustness,
reproducibility, maintainer burden, and user experience.

### 06 — Recommended strategy + roadmap
The recommendation, justified from chapters 03–05, expressed as a **phased,
actionable roadmap** maintainers could convert into issues/PRs. The prototype
phase is clearly marked as follow-up (not built here).

## Methodology

- **Primary source = the pinned submodule.** Every claim about current behavior
  cites a specific file (and line/section) in `external/Cytnx/`.
- **Distribution reality check.** Inspect the actual GitHub Actions workflows and
  any published artifacts (PyPI, conda-forge / anaconda) to state what *is*
  shipped today, not just what the build *can* do.
- **Reference projects cited to their own configs.** SciPy/CuPy claims are
  grounded in their public build/packaging files and docs, distilled into named,
  transferable patterns — not vague praise.
- **Diagrams** for the two things prose handles badly: the dependency graph, and
  the import-time library-resolution flow.

## Scope

### In scope
- Analysis of build-time **and** import-time deployment problems (one story).
- The required libraries: hptt, OpenBLAS (and MKL as the alternative), CUDA
  toolkit, cuTENSOR, cuQuantum, ARPACK, pybind11, and Boost.
- SciPy and CuPy as reference projects.
- Evaluation across build-from-source, pip, conda, pixi, and uv.
- A maintainer-facing recommendation and phased roadmap.

### Platform focus
- **Linux + CUDA** — primary GPU story (where cuTENSOR / cuQuantum / CUDA-toolkit
  pain concentrates).
- **macOS** — first-class CPU story: Apple Silicon / arm64, Accelerate vs.
  OpenBLAS, the hptt ARM variant, and the "no CUDA on Mac" GPU implication.
- **Windows** — noted where it materially differs; not a full deep-dive.

### Out of scope
- Building any working prototype configuration — that is the marked follow-up
  phase.
- Modifying the Cytnx submodule or opening upstream PRs — this repo *proposes*;
  acting on it is separate.
- Performance benchmarking or tuning of the dependencies — this is about
  *shipping* them, not making them fast.

## Success criteria
- Every required dependency (hptt, OpenBLAS/MKL, CUDA toolkit, cuTENSOR,
  cuQuantum, ARPACK, pybind11, Boost) is analyzed for both build-time discovery
  and import-time resolution (noting where a dependency is build-time-only).
- Current-state claims are cited to the pinned Cytnx submodule and to actual
  published artifacts / CI.
- SciPy and CuPy patterns are concrete and transferable, not generic praise.
- The channel evaluation yields a single, justified recommendation.
- Chapter 06 is actionable enough for a maintainer to open issues/PRs directly
  from it.

## Follow-up (not this project)
A later phase builds and validates prototype deployment configurations (e.g., a
working `pixi.toml`, a conda recipe, pip/uv wheel setup) demonstrating the
recommended strategy. Chapter 06 defines this phase; it is not implemented here.
