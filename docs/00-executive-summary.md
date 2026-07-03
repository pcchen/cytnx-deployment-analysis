# 00 — Executive Summary

Reference commit: Cytnx `v1.1.0` (`8d96d928d22cc176615e97cafdb7a3cef66cb732`), pinned as the read-only submodule at `external/Cytnx`.

## The problem

Cytnx already builds and publishes working CPU wheels to PyPI, but its GPU stack (CUDA/cuTENSOR/cuQuantum) has no CI coverage and no supported binary at all, and its dependency discovery relies on ambient search paths that fail silently at build time and replay as import-time crashes ([current-state audit](01-current-state-audit.md); [core problems](03-core-problems.md)). The result is a distribution story that works by accident for one CPU configuration and is undocumented, unreproducible, or entirely absent everywhere else ([dependency taxonomy](02-dependency-taxonomy.md)).

## The recommendation

Ship a **two-channel strategy split by dependency tier** — do not force one channel to do everything ([recommended strategy](06-recommended-strategy.md) §6.1, distilling the [channel evaluation](05-channel-evaluation.md) §5.4 matrix):

- **CPU (OpenBLAS/ARPACK): pip/wheels as the primary channel, with uv as a zero-cost drop-in on top.** This is the already-working, already-CI-built, near-zero-maintenance story ([current-state audit](01-current-state-audit.md) §1.2; [core problems](03-core-problems.md) §3.2).
- **GPU (CUDA/cuTENSOR/cuQuantum) plus anyone needing the most robust runtime resolution: the conda-forge ecosystem as the primary channel, with pixi as an optional reproducibility layer.** The entire proprietary GPU stack already exists as conda-forge packages, and the solver structurally replaces the ambient-path fragility diagnosed in [core problems](03-core-problems.md) with declared, versioned dependencies ([dependency taxonomy](02-dependency-taxonomy.md) §2.1; [channel evaluation](05-channel-evaluation.md) §5.4).
- **Build-from-source is the fallback, not a channel** — the escape hatch for unsupported configurations, not a recommended install path ([channel evaluation](05-channel-evaluation.md) §5.2).

By platform: **Linux + CUDA** has exactly one viable home (conda-forge); **macOS** stays on pip/wheels, which already ships correct `@loader_path`-relative wheels ([recommended strategy](06-recommended-strategy.md) §6.1; [core problems](03-core-problems.md) §3.2). The reference precedents for both channels are SciPy's BLAS-selection and wheel-repair playbook and CuPy's per-CUDA-version packaging playbook ([reference projects](04-reference-projects.md)).

## The 2–3 highest-impact changes

Each is prerequisite to trusting any packaging layered on top ([recommended strategy](06-recommended-strategy.md) §6.2):

1. **Fix the dead-code `*_FOUND TRUE` bug** in `FindCUTENSOR.cmake` / `FindCUQUANTUM.cmake` so a misconfigured GPU build fails loudly at configure time instead of silently at link/import time (effort S).
2. **Add a relocatable `$ORIGIN` entry to the Linux `INSTALL_RPATH`** (and drop the unexpanded `~`), bringing Linux to parity with the macOS `@loader_path` branch that already works (effort S–M).
3. **Add CI coverage for at least one `USE_CUDA=ON` configuration** — today the entire GPU path is exercised only by humans, never validated (effort M–L).

## Follow-up phase (out of scope here)

Actually authoring and validating the concrete distribution configs — a working `pixi.toml`, a submitted conda-forge feedstock, and hardened pip/uv wheel CI — is a deliberate **follow-up project, out of scope** for this analysis repository, which diagnoses and recommends rather than builds and ships ([recommended strategy](06-recommended-strategy.md) §6.4).
