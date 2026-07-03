# Wheel-Artifact Efficiency Chapter — Design Spec

**Date:** 2026-07-03
**Repo:** `github.com/pcchen/cytnx-deployment-analysis` (branch `build-analysis-doc` = `main`)
**Status:** Approved design, ready for implementation planning
**Motivating issue:** [Cytnx-dev/Cytnx#947](https://github.com/Cytnx-dev/Cytnx/issues/947)

## Purpose

Extend the existing Cytnx deployment analysis with a new, first-class chapter on
**wheel-artifact efficiency** (size vs. PyPI's storage quota) — an axis the
current document does not cover. Issue #947 shows Cytnx 1.1.0 is ~2.6 GB across
42 wheels against PyPI's 10 GB per-project quota (~3–4 releases of headroom), and
the current recommendation does not address it. This project folds #947's
findings into the document so it stays authoritative, keeping the analysis
(not prototypes) as the deliverable.

The two problems the document already covers (build-time discovery, import-time
resolution) are a **correctness** axis. Wheel-artifact efficiency is a **size**
axis — orthogonal, and currently the maintainers' most deadline-driven concern.

## Deliverable

A new chapter plus surgical, cross-referenced edits that keep the rest of the
document consistent. Reference point stays **Cytnx v1.1.0** (the release #947
measured; the pinned submodule at `external/Cytnx`,
`8d96d928d22cc176615e97cafdb7a3cef66cb732`).

### Files touched
- **`docs/07-wheel-artifact-efficiency.md`** (new) — the chapter.
- **`docs/02-dependency-taxonomy.md`** — size/redistribution notes on the
  OpenBLAS, ARPACK, and `libcytnx.a`/hptt entries, each cross-referencing ch.07.
- **`docs/06-recommended-strategy.md`** — insert a **Phase 0 (immediate: quota
  triage)** ahead of the existing phases; surface the C++-SDK-exclusion fix in
  §6.2's highest-impact changes.
- **`docs/00-executive-summary.md`** — one sentence flagging the quota wall,
  pointing to ch.07.
- **`README.md`** — add chapter 07 to "How to read this" (optional half-line in
  the TL;DR).

## Chapter 07 outline

### 7.1 The problem — a different axis
PyPI storage quota vs. current footprint: ~2.6 GB across 42 wheels against the
10 GB cap, ~3–4 releases of headroom (cite #947). Frame explicitly as *size*,
orthogonal to ch.03's correctness axis. Note the single-wheel spot check that
corroborates the two root causes.

### 7.2 Root cause 1 — the C++ SDK shipped inside the Python wheel
`libcytnx.a` + `include/` headers + CMake config are packaged into every wheel
because the `cytnx` STATIC target's install tree is what scikit-build-core
packages (cite the submodule install lines + the STATIC target). **LTO
amplifier:** Linux slim-LTO leaves GIMPLE IR bytecode in the archive instead of
machine code (cite the `CMakeLists.txt:223-231` LTO block — the same lines ch.01
cited for the macOS caveat, now read for size), corroborated by #947's
`Svd.cpp.o` 880 KB-IR-vs-199 KB-machine measurement. Spot check: `libcytnx.a`
≈ 70% of packed size.

### 7.3 Root cause 2 — double OpenBLAS on manylinux
cytnx links the *serial* OpenBLAS; the manylinux `arpack` package links the
*threaded* one; both are `DT_NEEDED`, so `auditwheel` vendors both. musllinux is
unaffected (single variant). Spot check: two OpenBLAS `.so`s present.

### 7.4 The downstream-consumer tradeoff
Cytnx ships headers + `.a` *deliberately* (documented via
`cytnx.__cpp_include__`/`__cpp_lib__` in `install.rst`), so a fix has a real
cost. Peer approaches (cite #947's comparison): **numpy** (tiny scoped `.a` +
runtime `PyCapsule` API), **scipy** (none), **torch** (shared libs). Design
space: exclude from the wheel and serve the C++ SDK separately
(sdist/conda/dedicated package), numpy-style scoping, or a split package.

### 7.5 Build-matrix breadth
42 wheels ≈ platforms × Python versions; note the multiplier and whether it is
reducible (stable-ABI/`abi3`, trimming Python versions) as a second-order lever
on total footprint. Diagnosis only — no abi3 migration is designed here.

### 7.6 Our own recommendation's blind spot
If GPU wheels ever land on PyPI via the per-CUDA scheme (ch.06 / ch.04 C1), the
per-CUDA multiplier × already-bloated wheels would hit the quota fast — so the
size fixes are a *prerequisite* to that, and moving GPU to conda-forge (ch.06)
also protects the quota. Honest self-critique of the existing recommendation.

### 7.7 Fixes → roadmap
Three concrete fixes, each with rough effort and an acceptance signal, feeding
ch.06's new Phase 0:
1. Exclude the `ARCHIVE`/headers/CMake-config from the Python wheel
   (scikit-build-core install-component exclusion).
2. Fix the LTO bloat so the shipped archive is not unreduced IR.
3. Dedupe OpenBLAS by aligning the `arpack` and cytnx builds to one variant.
Names the separate "fix project" (upstream PRs) as follow-up, out of scope here.

## Surgical edits to existing chapters

**`docs/02-dependency-taxonomy.md`** — additive notes only, no restructuring:
- OpenBLAS profile: manylinux carries two OpenBLAS copies (serial + threaded)
  → ch.07 §7.3.
- ARPACK profile: the manylinux `arpack` package links the threaded OpenBLAS,
  forcing the second copy → ch.07 §7.3.
- `libcytnx.a`/hptt note: extend to note `libcytnx.a` is shipped into the wheel,
  the dominant size cost → ch.07 §7.2.

**`docs/06-recommended-strategy.md`:**
- Insert **Phase 0 — immediate quota triage** ahead of existing phases: the three
  §7.7 fixes with effort + acceptance signals, framed as the most time-sensitive
  work. Existing phases renumber but keep their content.
- §6.2: add "exclude the C++ SDK from the wheel (+ LTO bloat / OpenBLAS dedupe)"
  to the highest-impact changes, noted as the most deadline-driven.

**`docs/00-executive-summary.md`** — one sentence on the quota wall → ch.07.

**`README.md`** — add `8. [Wheel-artifact efficiency](docs/07-wheel-artifact-efficiency.md)`
to "How to read this"; optional half-line in TL;DR.

## Methodology
- **#947 is cited as the source investigation** for all size measurements and the
  numpy/scipy/torch comparison — attributed, not silently absorbed.
- **Every causal/CMake claim is independently verified** against the pinned
  v1.1.0 submodule (the STATIC target and its `ARCHIVE`/headers/CMake-config
  install; the LTO block; the ARPACK link line; the OpenBLAS linkage) with exact
  `file:line` citations — same rigor as chapters 01–06.
- **One `cp312` manylinux wheel is downloaded and extracted** to independently
  corroborate the two headline numbers (`libcytnx.a` ≈ 70% of packed size; two
  OpenBLAS `.so`s). If the download fails, the chapter says so plainly and leans
  on #947's measurements rather than inventing figures.
- Fixes are **diagnosed and recommended**, not prototyped.

## Scope

### In scope
The two root causes; the downstream-consumer tradeoff; build-matrix breadth; the
per-CUDA/PyPI blind spot; the Phase-0 fixes; and keeping ch.02/06/00/README
consistent.

### Out of scope
- **Implementing** any fix — no `pyproject.toml`/CMake patches, no PR to Cytnx.
  ch.07 §7.7 names that as a separate follow-up project.
- **Re-measuring the full 42-wheel set** — one wheel corroborates; full
  re-measurement is redundant with #947.
- **Redesigning the build matrix** — §7.5 notes breadth as a lever but designs no
  abi3 migration.
- Windows stays noted-where-it-differs (ships no wheels today).

## Success criteria
- Chapter 07 covers all sections 7.1–7.7 with #947 attributed and every
  CMake/causal claim cited to the pinned submodule.
- The single-wheel spot check is performed and its two headline numbers reported
  (or its failure stated plainly).
- ch.02 (three notes), ch.06 (Phase 0 + §6.2), ch.00 (one sentence), and README
  (chapter link) are updated and internally consistent with ch.07.
- Whole-document invariants still hold: 8 chapters (`docs/00`–`07`), no
  placeholders, all internal links resolve.
- No fix is implemented; the follow-up fix project is named, not built.
