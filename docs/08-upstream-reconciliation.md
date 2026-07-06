# 08 — Upstream Reconciliation & Status Board

> Upstream tracking issue: Cytnx-dev/Cytnx#970
>
> Forward-looking task view: [chapter 09 — Action Plan & Item Tracker](09-action-plan.md)

**Reconciled against:** `Cytnx-dev/Cytnx` `master` @ `32079a4` on 2026-07-05.

Maps every ch.06 (Phases 0–3) and ch.07 roadmap item to its current upstream
reality. Classifications: `covered` (an open/merged PR already does it) ·
`dissolved` (the v1.1.0 finding no longer holds on `master`) · `gap` (real,
actionable now) · `gap-decision-coupled` (real gap whose fix depends on the
#969 packaging-model decision) · `superseded` (subsumed/reframed by a stronger
upstream artifact, chiefly #969).

All "Verified on `master`?" findings below were produced by running `git -C
../Cytnx-upstream grep/diff` against the real, shallow clone of upstream
`master` at commit `32079a4` (2026-07-01), and by inspecting `gh pr diff
<n> -R Cytnx-dev/Cytnx` for the cited PRs. `external/Cytnx` (the stale v1.1.0
submodule) was never consulted for line numbers or current-state claims.

## Reconciliation table

| Item | Our ref | Upstream | Status | Verified on `master`? | macOS-verifiable? | Classification |
|------|---------|----------|--------|-----------------------|-------------------|-----------------|
| 0.1 Exclude C++ SDK from wheel | ch.07 §7.2, §7.7 | PR #967, issue #947 | PR #967 open-PR; issue #947 open-issue | Not on master: `pyproject.toml` has no `install.components` key, and `CMakeLists.txt`'s `ARCHIVE` install rule is still tagged `COMPONENT libraries` (not `Development`), so a wheel build still ships `libcytnx.a`. PR #967's diff retags that rule `COMPONENT Development` and adds `install.components = ["libraries"]` to `pyproject.toml`, which is what actually excludes the static lib/headers/CMake config from the wheel. **Locally confirmed 2026-07-05** (see "Local build verification" below): a manylinux_2_28 aarch64 wheel built from #967's branch contains no `libcytnx.a`, headers, or CMake config. | yes (cibuildwheel Docker / native `pip wheel .` content check) | covered (open PR #967) |
| 0.2 Linux slim-LTO `.a` GIMPLE-IR bloat | ch.07 §7.2 | (obviated if #967 stops shipping `.a`?) | none (no dedicated tracked issue; riding on #967) | On master, `CMakeLists.txt:227` already disables IPO/LTO on `APPLE` (breaks the `libcytnx.a` link otherwise) but leaves `CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE` unconditionally on Linux, so the GIMPLE-IR-bloated `.a` is still produced there today. Since 0.1 confirms `libcytnx.a` is still shipped in the wheel on master (not yet fixed by #967), the LTO-bloat symptom is still live; once #967 lands and the wheel stops carrying `libcytnx.a` at all, this item's target artifact disappears rather than needing its own fix. | n/a (superseded by 0.1/#967 landing, not independently fixable) | superseded (subsumed by #967 — the fix is removing the artifact, not de-bloating it) |
| 0.3 Dedup OpenBLAS across manylinux arpack+cytnx builds | ch.07 §7.3, §7.7 | issue #947 | issue #947 open-issue | **Correction to the brainstormed anchor:** this is not merely referenced by #947, it is already implemented in open PR #967. `gh pr diff 967` adds a `file(GET_RUNTIME_DEPENDENCIES ...)` + `check_library_exists(... LAPACKE_dgesdd ...)` block to `CytnxBKNDCMakeLists.cmake` that switches `BLAS_LIBRARIES`/`LAPACK_LIBRARIES`/`LAPACKE_LIBRARIES` to match arpack's own OpenBLAS build when they differ, specifically to stop both pthread and serial OpenBLAS builds from being vendored into manylinux wheels. Master itself has no such logic (`git grep -n "GET_RUNTIME_DEPENDENCIES"` in the clone returns no matches). **Locally confirmed working 2026-07-05** (see "Local build verification" below): on a real manylinux_2_28 aarch64 build the switch fires by design and the repaired wheel vendors exactly one OpenBLAS. | yes (cibuildwheel Docker — only activates where a system `arpack` package coexists with cytnx's own OpenBLAS resolution, i.e. manylinux) | covered (open PR #967) |
| 1.1 `*_FOUND TRUE` dead-code bug (FindCUTENSOR/FindCUQUANTUM) | ch.03 §3.1, ch.02 §2.2 | PR #950, issues #945/#946 | PR #950 open-PR; #945 open-issue; #946 open-issue | Not on master: `cmake/Modules/FindCUTENSOR.cmake` and `FindCUQUANTUM.cmake` both still end with an unconditional `set(CUTENSOR_FOUND TRUE)` / `set(CUQUANTUM_FOUND TRUE)` regardless of whether `find_library()` actually resolved anything (confirmed via `git grep -nE "_FOUND"` on both files). PR #950's diff replaces both with `include(FindPackageHandleStandardArgs)` + `find_package_handle_standard_args(...)`, which fails correctly on an unresolved library. **Update 2026-07-05: PR #950 will NOT be merged — it is to be split into several focused PRs (see "PR #950 split" below); this item is `covered` only once split PR-1 (finder `*_FOUND` fix) lands.** | no (Linux+CUDA — `USE_CUTENSOR`/`USE_CUQUANTUM` are `cmake_dependent_option(... "USE_CUDA" OFF)`, unreachable without a CUDA toolkit) | gap (was in PR #950; pending split PR-1) |
| 1.2 Add `$ORIGIN`/remove unexpanded `~` in INSTALL_RPATH | ch.03 §3.2 | (check if dissolved on master; see #948) | n/a (no direct tracked issue; #948 open-issue is a related but distinct Linux CUDA-runtime rpath concern) | **Dissolved for the primary defect.** Master's macOS `INSTALL_RPATH` block (`CMakeLists.txt:432-434`) is `"@loader_path;@loader_path/../../../lib;${CMAKE_INSTALL_PREFIX}/lib"` — `@loader_path` now leads, so the extension is relocatable without depending on the third entry at all; `git grep -n "~"` over `CMakeLists.txt` shows no literal `~` in the RPATH block itself. Caveat found during verification: `CMAKE_INSTALL_PREFIX`'s repo default is the literal string `~/.local/cytnx` (`CMakeLists.txt:166`), and CMake does not shell-expand `~` in variable substitution — so a bare `cmake --install` with no `-DCMAKE_INSTALL_PREFIX` override would still embed an unexpanded `~` in that trailing fallback RPATH entry. This is inert in practice only because `@loader_path` is tried first and normally succeeds; it is not a full closure of the original defect, just a demotion of it to an unused fallback. | yes (native) | dissolved (primary defect resolved by `@loader_path`; latent fallback-entry `~` noted but non-blocking) |
| 1.3 Remove stale legacy `CUDA_*_LIBRARY`/`linkflags.tmp` dead block | ch.02 §2.2 | issue #949, PR #950 | issue #949 open-issue; PR #950 open-PR | **Not covered by PR #950 — contradicts the brainstormed "1.1/1.3→covered" anchor for 1.3 specifically.** `gh pr diff 950` only touches the two `message(STATUS ...)` display lines (`CUDA_VERSION_STRING`→`CUDAToolkit_VERSION`, `CUDA_TOOLKIT_ROOT_DIR`→`CUDAToolkit_BIN_DIR`), which fixes issue #949's blank-value *symptom*. It does not touch `CytnxBKNDCMakeLists.cmake:243-254`, which still appends `CUDA_cusolver_LIBRARY`, `CUDA_curand_LIBRARY`, `CUDA_cublas_LIBRARY`, `CUDA_cudart_static_LIBRARY`, `CUDA_cudadevrt_LIBRARY`, `CUDA_cusparse_LIBRARY` into `linkflags.tmp`. None of these legacy `FindCUDA`-style variables are ever `set()`/`find_library()`'d anywhere in the repo (`git grep -n "CUDA_cusolver_LIBRARY"` only matches the two dead-code lines that reference it), so on master this block always appends empty strings — genuinely dead code, still present, and not touched by PR #950's diff. | no (Linux+CUDA — block only executes `if(USE_CUDA)`) | gap |
| 1.4 Correct stale `adv_install.rst` | ch.01 §1.4, ch.02 §2.2 | docs edits in PR #950 & #967 (contested) | PR #950 open-PR; PR #967 open-PR | The compiling-options table (`docs/source/adv_install.rst:224-240`) on master lists `-DCMAKE_INSTALL_PREFIX` (shown with default `/usr/local/cytnx`, while `CMakeLists.txt:166`'s actual default is `~/.local/cytnx` — the table is factually wrong on master today), `-DUSE_MKL`, `-DUSE_OMP`, `-DUSE_CUDA`, `-DUSE_HPTT`, but omits `-DUSE_CUTENSOR`, `-DUSE_CUQUANTUM`, and `-DBACKEND_TORCH` even though all three are real options declared at `CMakeLists.txt:168-183`. Neither PR #950 (adds CUDA 12/13 selection + tarball runtime-path prose elsewhere in the same file) nor PR #967 (rewrites the C++-interop sections of `install.rst`/`adv_install.rst`) touches this options table in their current diffs — both add adjacent prose but leave the actual defect (the stale/incomplete table) unfixed. macOS-vs-MKL OpenBLAS guidance is present in prose (`adv_install.rst:26` `[MKL]` section) but not cross-referenced from the options table either. | yes (native — docs-only) | gap |
| 1.5 Installed CUDA extension not rpath-pinned to build toolkit | — | issue #948 | issue #948 open-issue | Not on master and inherently Linux+CUDA-scoped: `CMakeLists.txt`'s only `INSTALL_RPATH` block is gated `if(APPLE)` (lines 429-439); no equivalent Linux CUDA-extension rpath-to-build-toolkit pinning exists anywhere in the clone. Issue #948 describes the installed extension instead falling back to loading whatever CUDA runtime the loader/`ldconfig` resolves (e.g. an apt-installed copy), which can silently version-skew against the toolkit it was built with. | no (Linux+CUDA) | gap-decision-coupled (real gap; also not verifiable without Linux+CUDA hardware, and how the CUDA runtime libs get vendored/pinned depends on the still-undecided #969 packaging model) |
| 2.1 Document `pip install cytnx` / `uv add cytnx` as first-class | ch.01 §1.3-1.4, ch.05 §5.4 | — | none (untracked upstream) | Real, verified gap: `git grep -nE "pip install\|uv add\|PyPI" docs/source/install.rst` returns nothing. `install.rst` opens with "we recommend user to use anaconda/miniconda" and its macOS section literally says "Please build from source, currently 0.9+ does not have conda package support" — yet `.github/workflows/release_pypi.yml` already builds and publishes `cytnx` to production PyPI via OIDC trusted publishing on every `v*` tag (its own header comment documents the stable install command as `pip install cytnx`), including macOS wheels via cibuildwheel. The capability exists and is exercised by CI today; it is simply never mentioned in the user-facing install docs. | yes (native — docs-only, directly cross-checkable against `release_pypi.yml`) | gap |
| 2.2 SciPy S1/S2: pinned OpenBLAS/ARPACK build-only source + documented auditwheel/delocate | ch.04 §4.1-4.3 | — | none | Mechanism partially present, documentation absent: `pyproject.toml:100` pins `manylinux-x86_64-image = "manylinux_2_28"`; `[tool.cibuildwheel]` (lines 95-113) has no `repair-wheel-command` override, so cibuildwheel's default `auditwheel repair` (Linux) / `delocate` (macOS) run unmodified (confirmed by absence of any override key). None of this — the manylinux image choice, the OpenBLAS/ARPACK dedup rationale (see 0.3, in PR #967), or the auditwheel/delocate repair step — is documented anywhere under `docs/source/*.rst` for downstream packagers. | yes (native — docs-only; the delocate half is directly inspectable on macOS) | gap |
| 2.3 Move conda from `kaihsinwu` to conda-forge feedstock; fix version lag | ch.01 §1.2/§1.4 | — | none | Not done: `docs/source/install.rst:56,62` still instruct `conda install -c kaihsinwu cytnx` and `-c kaihsinwu cytnx_cuda`; `.github/workflows/conda_build_release.yml:69` still uploads via `anaconda -t $TOKEN upload -u kaihsinwu`. `conda_build/meta.yaml` is the personal-channel recipe, not a conda-forge feedstock; no feedstock exists to check version lag against. The CPU-only package's move to conda-forge looks plausibly independent of any CUDA decision, but item 2.3 as scoped also covers fixing the broken `cytnx_cuda` channel reference (see 3.5), whose correct long-term shape depends on #969. | yes (native, for the recipe/doc-inspection half) | gap-decision-coupled (CPU feedstock migration is real but its packaging shape, especially for any CUDA variant, is entangled with #969) |
| 2.4 macOS Accelerate-linked BLAS option (optional) | ch.05 §5.3, ch.04 §4.1 | — | none | Not implemented: the only `APPLE`-conditioned branches in `CMakeLists.txt` are LTO-disable (line 227), the pycytnx `-undefined dynamic_lookup` link option (line 420), and the RPATH block (line 429). `git grep -niE "Accelerate framework|vecLib|-framework Accelerate|BLA_VENDOR.*Apple"` over the clone returns no matches — there is no `Accelerate`/`vecLib` framework option (a bare `git grep -ni accelerate` matches only the unrelated "accelerate tensor transpose" HPTT help string at `CMakeLists.txt:170`). macOS builds resolve BLAS via the generic `find_package(BLAS)`, using Homebrew OpenBLAS per cibuildwheel's `CMAKE_PREFIX_PATH`, with no Accelerate-linked alternative. | yes (native) | gap |
| 3.1 CI workflow building/importing `openblas-cuda` on a CUDA runner | §6.2 item 3, ch.01 §1.1-1.2 | — | none | Real gap: the `openblas-cuda`/`mkl-cuda` CMake presets already exist (`CMakePresets.json:50,62,95,97,126`), but no workflow under `.github/workflows/` builds or imports them on a CUDA runner (`git grep -l "openblas-cuda" .github/workflows/` → no matches). The full workflow list (`ci-cmake_tests.yml`, `ci-downstream-find-package.yml`, `conda_build.yml`, `conda_build_release.yml`, `release_pypi.yml`, `docs.yml`, `clang-format-check.yml`, `coverity-scan.yml`, `version-consistency.yml`) contains no CUDA runner at all. | no (Linux+CUDA runner required) | gap |
| 3.2 CuPy C2 ordered search chain in FindCUTENSOR/FindCUQUANTUM | ch.03 §3.1, ch.04 §4.2 | issue #946, PR #950 | issue #946 open-issue; PR #950 open-PR | Not on master: master's finders use a single hard-coded `CUTNLIB_DIR` guess selected per CUDA major version (`10.2`/`11.0`/`11`/`12`) with `NO_DEFAULT_PATH`, i.e. one fixed path, not an ordered chain. PR #950's diff replaces this with `PATH_SUFFIXES lib lib/${CUDAToolkit_VERSION_MAJOR}`, trying both the flat cuTENSOR-2.x tarball layout and the legacy per-major/apt layout in order, plus explicit cuTENSOR>=2.0 version-header parsing with a `FATAL_ERROR` below 2.0. **Update 2026-07-05: PR #950 will NOT be merged — being split (see "PR #950 split" below); this item is `covered` only once split PR-2 (FindCUTENSOR layout) lands.** | no (Linux+CUDA — cuTENSOR/cuQuantum are Linux-only NVIDIA libraries) | gap (was in PR #950; pending split PR-2) |
| 3.3 conda-forge GPU deps (cudatoolkit/cutensor/cuquantum) — CuPy C3 | ch.02 §2.1, ch.04 §4.2 | — | none | Not present: `conda_build/meta.yaml` defines one package (`cytnx`) with CPU-only requirements (`blas=*=mkl`/`blas=*=openblas` selectors only, no `cudatoolkit`/`cutensor`/`cuquantum`); `conda_build_release.yml`'s build matrix is `[ubuntu-latest, macos-15-intel, macos-latest]`, no CUDA runner. Issue #969's own body explicitly cites "Cytnx currently ships ... a single CPU-only package on conda" as context for its packaging-strategy investigation — i.e. #969 already subsumes this exact question on the conda side. | no (Linux+CUDA for build/test) | superseded (subsumed by #969's conda-side investigation) |
| 3.4 Per-CUDA-major wheels (`cytnx-cuda12x`) — CuPy C1 (optional) | ch.04 §4.2 | #969 | issue #969 open-issue | Not on master: `pyproject.toml` has no CUDA build variant, extra, or index configuration at all (independently confirmed and cited inside #969's own verified context). #969 is precisely evaluating this option (a PyTorch/JAX-style per-CUDA-major variant wheel) as one of several candidate strategies, with no decision recorded yet. | n/a (strategy decision, not a macOS-testable code change) | superseded (this item is literally one of the options #969 is deciding between) |
| 3.5 Reconcile advertised-but-unbuilt `cytnx_cuda` conda package | ch.01 §1.4 | — | none | Confirmed inconsistency: `docs/source/install.rst:62` tells users to run `conda install -c kaihsinwu cytnx_cuda`, but `conda_build/meta.yaml` defines only the CPU `cytnx` package and `conda_build_release.yml`'s build matrix has no CUDA runner or build step — no `cytnx_cuda` package is ever built or uploaded by current CI, so the documented command 404s today. | yes (native — this is a doc correction; verifiable by reading the recipe/workflow, no CUDA needed) | gap |
| S Packaging-model strategy (PyTorch vs JAX variants) | ch.04-06 | issue #969 (+#966) | issue #969 open-issue; issue #966 open-issue | Both open, undecided. #969 is an investigative report (its own text notes it was researched to inform "a future CUDA-enabled release") comparing PyTorch's GPU-by-default-on-Linux model against JAX's CPU-by-default/bolt-on-plugin model, verified against live PyPI JSON metadata and actual wheel contents, explicitly framed as "input to that design decision" — no decision or implementing PR exists yet. #966 (shared `libcytnx.so`, PyTorch-style) is referenced from PR #967's description as a specific follow-up under consideration for restoring C++-interop without the static-lib bloat. | n/a (strategy row) | gap (the packaging-model decision itself remains the open, actionable gap; #969 is progress toward closing it, not a resolution) |

## Local build verification (2026-07-05)

PR #967's two Phase-0 items (0.1 SDK exclusion, 0.3 OpenBLAS dedup) were verified
empirically, not just by diff-reading — the dedup half is exactly the part the PR
author flagged as untested. #967's code changes were applied onto the `master`
clone (docs hunks excluded — irrelevant to the build) and a wheel was built via
`cibuildwheel` on the real `quay.io/pypa/manylinux_2_28_aarch64` image
(`CIBW_BUILD=cp312-manylinux_aarch64`, native on the Apple-Silicon host).

Result — `cytnx-1.1.1-cp312-cp312-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl`:

- **OpenBLAS dedup fires by its intended mechanism.** Configure log:
  `Found BLAS: /usr/lib64/libopenblas.so` (serial, from `find_package(BLAS)`) →
  `Switching to arpack's own OpenBLAS build to avoid linking two copies:
  /lib64/libopenblasp.so.0` (pthread) → cytnx links the pthread build. The
  repaired wheel's `cytnx.libs/` holds **exactly one** OpenBLAS
  (`libopenblasp-r0-*.so`) alongside `libarpack`, `libgfortran`, `libgomp`.
- **SDK exclusion holds:** no `libcytnx.a`, no headers, no CMake config in the wheel.

Caveats: (1) **confirmed on both arches** — aarch64 (native) and x86_64 (QEMU,
~51 min), each vendoring a single OpenBLAS via the same switch. Earlier "aarch64
only" caveat resolved (2026-07-06).
(2) CMake emits its `file(GET_RUNTIME_DEPENDENCIES) ... in project mode ... not
recommended` warning during configure — non-fatal here, but the mechanism leans on
a code path CMake documents as install-time-only.

Reported upstream as a follow-up comment on PR #967 (2026-07-05). The review was
resolved positively — IvanaGyro confirmed the ARPACK/`BACKEND_TORCH` point (fixed
the commit message) and accepted the aarch64 build as validating the dedup logic.
A suggested single-OpenBLAS CI gate was **declined** (issue #971, closed "not
planned"): #967's goal is wheel-size reduction, not a hard guarantee, and a strict
"exactly one OpenBLAS" assertion is in fact incorrect — #967 intentionally allows
two variants when arpack's OpenBLAS lacks LAPACKE (the fallback branch).

## PR #950 split (2026-07-05)

PR #950 ("require GCC≥13, CUDA≥12, cuTENSOR≥2.0; harden CUDA finders") **will not
be merged as-is — it is to be split into several focused PRs.** It bundles six
separable changes; the split below covers 100% of #950. Until the split PRs land,
items **1.1** and **3.2** revert from `covered` to `gap`.

| Split PR | #950 content | Covers | Files | Notes |
|----------|--------------|--------|-------|-------|
| **PR-1** | `*_FOUND` derived from `find_package_handle_standard_args`; FindCUQUANTUM conditional libs | **1.1** / #945-#946 | FindCUTENSOR/FindCUQUANTUM.cmake | Smallest, pure correctness, no policy dep. **Best first.** |
| **PR-2** | FindCUTENSOR version-aware `lib/${MAJOR}` subdir; drop dead `lib/10.2`,`lib/11` | **3.2** / #946 | FindCUTENSOR.cmake | Same file as PR-1 → sequence after PR-1, or merge the two into one "finder fixes" PR. |
| **PR-3** | #949 diagnostics on modern `CUDAToolkit_*` — **extend to also remove the legacy `CUDA_*_LIBRARY`→`linkflags.tmp` block** #950 left out of scope | **#949 + item 1.3** | CytnxBKNDCMakeLists.cmake | One clean "kill legacy FindCUDA vars" PR closing both. |
| **PR-4** | Version-floor policy: CUDA≥12 (12&13), `CUDAToolkit` before `enable_language`, arch adapt (sm_70 only 12.x), cuTENSOR≥2.0, nvcc-not-found error | **#962** decision | CMakeLists.txt, CytnxBKNDCMakeLists.cmake | Biggest/opinionated; land **after #962 is explicitly decided**. |
| **(fold)** | GCC≥13 enforcement | — | CMakeLists.txt | **Fold into existing PR #961** (already requires GCC≥13 for `std::format`) — do not open a new PR; avoids a conflicting duplicate. |
| **PR-6** | `adv_install.rst`: CUDA≥12/cuTENSOR≥2.0 dep list + tarball `CUTENSOR_ROOT`/`LD_LIBRARY_PATH` note | **#948 doc workaround** | adv_install.rst | Docs-only, macOS-verifiable. |

Net: **5 new PRs (PR-1…4, PR-6) + fold GCC≥13 into #961** reproduce all of #950.
Recommended start: **PR-1** (isolated correctness fix; independent of the #962 and
#961 decisions).

## First-PR candidate

Gates (all required): (1) classification is `gap` or, in substance, a real gap
(`gap-decision-coupled` counts as "yes" here — it is still a genuine gap, it
just also fails gate 3 by definition); (2) macOS-verifiable; (3) not
`gap-decision-coupled` on #969.

| Item | (1) gap? | (2) macOS-verifiable? | (3) not #969-coupled? | Passes all? |
|------|----------|------------------------|------------------------|-------------|
| 0.1 SDK exclusion | no (covered) | — | — | ✗ |
| 0.2 LTO bloat | no (superseded) | — | — | ✗ |
| 0.3 OpenBLAS dedup | no (covered — corrected from brainstormed anchor) | — | — | ✗ |
| 1.1 `*_FOUND TRUE` bug | yes (gap — #950 splitting) | no (Linux+CUDA) | yes | ✗ (fails macOS gate) |
| 1.2 rpath `~` | no (dissolved) | — | — | ✗ |
| 1.3 legacy CUDA block | yes | no (Linux+CUDA) | yes | ✗ |
| 1.4 stale `adv_install.rst` options table | yes | yes (native) | yes | ✓ |
| 1.5 CUDA-runtime rpath (#948) | yes | no (Linux+CUDA) | no | ✗ |
| 2.1 pip/uv install docs | yes | yes (native) | yes | ✓ |
| 2.2 SciPy S1/S2 pins + docs | yes | yes (native) | yes | ✓ |
| 2.3 conda-forge move | yes | yes (partial) | no (CUDA-variant half coupled to #969) | ✗ |
| 2.4 macOS Accelerate BLAS option | yes | yes (native) | yes | ✓ |
| 3.1 CUDA CI runner | yes | no (Linux+CUDA runner) | yes | ✗ |
| 3.2 CuPy C2 search chain | yes (gap — #950 splitting) | no (Linux+CUDA) | yes | ✗ (fails macOS gate) |
| 3.3 conda-forge GPU deps | no (superseded) | — | — | ✗ |
| 3.4 per-CUDA-major wheels | no (superseded) | — | — | ✗ |
| 3.5 `cytnx_cuda` conda doc fix | yes | yes (native) | yes | ✓ |
| S packaging strategy | yes | n/a | yes (it's the referent, not coupled to itself) | ✗ (n/a is not "yes") |

Five items pass all three gates: 1.4, 2.1, 2.2, 2.4, 3.5. All five are real,
and none blocks on the #969 decision, so this is an honest multi-candidate
result rather than a single obvious gap — the choice among them comes down to
risk/impact, not gate eligibility.

**Selected first PR: 2.1 — document `pip install cytnx` / `uv add cytnx` as
first-class in `install.rst`.** Rationale: it is the highest-impact of the
five (today's docs actively steer macOS users toward building from source —
"currently 0.9+ does not have conda package support" — despite
`release_pypi.yml` already building and publishing tested macOS/Linux PyPI
wheels on every stable tag), it is a pure documentation addition with zero
code/build-system risk, and it requires no coordination with the two large
in-flight PRs (#950, #967 touch `adv_install.rst`'s CUDA/C++-interop
sections and `CMakeLists.txt`/`pyproject.toml`, not `install.rst`'s opening
recommendation). 1.4, 2.2, 3.5, and 2.4 remain good follow-on candidates
(1.4/2.2/3.5 could reasonably be folded into the same docs-accuracy pass; 2.4
is larger in scope — new CMake/Accelerate-framework integration work, not a
doc fix — and is better scheduled as its own PR rather than bundled into a
first, low-risk change).

If a Linux+CUDA-capable environment becomes available, 1.3 (remove the dead
legacy `CUDA_*_LIBRARY`/`linkflags.tmp` block) and 3.1 (wire the existing
`openblas-cuda` CMake preset into a CUDA-runner CI job) are the next-clearest
gaps — both are real and un-touched by any open PR, but neither can be
verified from this macOS clone.
