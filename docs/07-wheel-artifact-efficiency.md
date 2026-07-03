# 07 — Wheel-Artifact Efficiency

Reference commit: Cytnx `v1.1.0` (`8d96d928d22cc176615e97cafdb7a3cef66cb732`).
Size measurements are attributed to [Cytnx-dev/Cytnx#947](https://github.com/Cytnx-dev/Cytnx/issues/947);
causal/CMake claims are verified against the pinned submodule; the §7.1 spot
check was performed independently for this chapter, on 2026-07-03, against the
cp312 manylinux x86_64 wheel of the same `1.1.0` release.

## 7.1 The problem — a different axis

Every prior chapter in this document treats Cytnx's packaging problem as a
*correctness* problem: does the right dependency get found at build time, and
does the right one get resolved at import time (ch.03 §3.1–§3.2)? This chapter
asks a question ch.03 never raises — not "is the artifact correct?" but "is
the artifact too big?" — and the answer bears directly on whether the project
can keep publishing releases at all.

[Cytnx-dev/Cytnx#947](https://github.com/Cytnx-dev/Cytnx/issues/947) reports
that the `cytnx` 1.1.0 release on PyPI totals **~2.6 GB across 42 wheel
files**, against PyPI's per-project storage quota of **10 GB** (#947). At the
current per-release size, the issue estimates the project has room for
**roughly 3–4 more releases** before hitting the quota (#947). This is not a
correctness failure — every one of those 42 wheels installs and imports
correctly (ch.01 §1.2's published-artifact check found no gaps in the
platform/Python matrix) — it is a resource-exhaustion problem with a fixed,
external ceiling that ch.01–ch.06 never modeled. Ch.03's throughline was
"build-time discovery is where ambiguity is introduced, import-time
resolution is where it's exposed"; this chapter's throughline is structurally
parallel but orthogonal: *what CMake installs into the wheel's build tree is
where bloat is introduced, and PyPI's storage quota is where it's exposed* —
two independent failure axes over the same artifact.

**One-wheel spot check.** For this chapter, one cp312 manylinux x86_64 `cytnx`
1.1.0 wheel was downloaded directly from PyPI
(`cytnx-1.1.0-cp312-cp312-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl`,
97,031,667 bytes on disk) into the scratchpad (not committed to this repo) and
inspected with `zipfile`/`unzip -v`. Measured directly from that file: `lib64/
libcytnx.a` is 104,582,084 bytes uncompressed / 64,138,195 bytes packed inside
the wheel, accounting for **66.1% of the wheel's total packed payload**
(64,138,195 of 97,018,745 summed compressed bytes across all wheel members);
exactly **two** OpenBLAS shared objects are present,
`cytnx.libs/libopenblas-r0-11edc3fa.3.15.so` (37,105,401 bytes uncompressed /
10,951,109 packed) and `cytnx.libs/libopenblasp-r0-59ffcd50.3.15.so`
(38,683,921 uncompressed / 11,254,581 packed). Both the uncompressed
`libcytnx.a` size and the two OpenBLAS `.so` sizes match #947's cited figures
to within rounding; the measured packed-share of `libcytnx.a` (66.1%) is a few
points below #947's cited "~70%" headline, most likely because the two
calculations divide by different denominators (this chapter divides by the
sum of every wheel member's compressed size; #947 does not state its exact
denominator) — the underlying conclusion is unaffected either way:
`libcytnx.a` alone accounts for roughly two-thirds to three-quarters of the
wheel's packed weight, on one measurement or the other. §7.2 and §7.3 below
walk the two root causes this spot check corroborates.

## 7.2 Root cause 1 — the C++ SDK shipped inside the Python wheel

**What is packaged, and why.** Cytnx's core is built as a static library,
`add_library(cytnx STATIC)` (`external/Cytnx/CMakeLists.txt:233`, already
established as the link target for the `pycytnx` Python extension in ch.03
§3.2). At install time, that target's `ARCHIVE` output — the compiled
`libcytnx.a` — and its `PUBLIC_HEADER` output are installed together in one
`INSTALL(TARGETS cytnx EXPORT cytnx_targets ...)` block that does carry
`COMPONENT libraries`/`COMPONENT Development` tags
(`external/Cytnx/CMakeLists.txt:457–467`), but nothing sets
scikit-build-core's `install.components` option in `pyproject.toml`, so those
tags are never exercised and the whole install tree is packaged regardless.
The same install pass
also emits a full CMake package-config export — `install(EXPORT cytnx_targets
...)` (`external/Cytnx/CMakeLists.txt:477–481`) — and copies the entire
`include/` tree (`install(DIRECTORY include/ ... FILES_MATCHING PATTERN
"*.h*")`, `external/Cytnx/CMakeLists.txt:482–485`). None of this is
CI-specific or wheel-specific code: it is the *one* install tree Cytnx's
CMake produces for every consumer, C++ or Python. Because Cytnx's Python
wheels are built by `scikit-build-core` driving this exact same CMake install
step (ch.01 §1.3's "`pip install .` ... driven by scikit-build-core",
`external/Cytnx/pyproject.toml`), the wheel scikit-build-core assembles is
simply "whatever `cmake --install` produced," and nothing in that tree is
marked as Python-runtime-only versus C++-development-only. The result: a
Python user who only ever calls `import cytnx` still downloads the static
archive, every header, and the CMake package-config files a C++ consumer
would need to `find_package(Cytnx)` — none of which the Python extension
module (`cytnx*.so`, ch.03 §3.2) needs at import time (#947).

**The LTO amplifier.** The size of `libcytnx.a` itself is inflated further on
Linux specifically. `CMakeLists.txt:223–231` sets
`CMAKE_INTERPROCEDURAL_OPTIMIZATION` — CMake's LTO switch — `FALSE` on Apple
and `TRUE` everywhere else:

```cmake
IF (APPLE)
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)
ELSE ()
  set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
ENDIF ()
```

Ch.01 §1.1 already cites this same block for a different reason — it is the
macOS static-link-failure caveat (a build-breakage risk). For size purposes,
the consequence runs the other way: on Linux, GCC's default "slim" LTO mode
means every `.o` placed inside the static archive holds only GIMPLE
intermediate-representation bytecode (`.gnu.lto_*` sections), not compiled
machine code — and because that IR is archived as-is rather than passed
through a final LTO link/reduction step before being installed, it never gets
collapsed back down to native code. #947 confirms this directly by extracting
the identical source object, `Svd.cpp.o`, from both the Linux and macOS
builds: **880 KB of pure LTO bytecode with an empty `.text` section on
Linux, versus 199 KB of real machine code on macOS** — a 4.4× size
difference for the same compiled unit, compounding across all ~180 object
files in the archive (#947). This is why the spot check above found
`libcytnx.a` occupying roughly two-thirds of the manylinux wheel while
the parallel macOS-arm64 figure (#947) is comparatively small
(27.2 MB uncompressed / 4.7 MB packed): the same static-archive design choice
is present on both platforms, but only Linux's LTO configuration turns it
into a bytecode dump rather than a compiled library.

**Per-wheel sizes.** #947's measured packed sizes for the six wheel variants
built from the same cp312 build (representative of the other supported
CPython versions, which are "near-identical" per #947):

| Wheel | Packed size (#947) |
|---|---|
| macOS arm64 | 19.46 MB |
| macOS x86_64 | 25.10 MB |
| manylinux x86_64 | 92.54 MB |
| manylinux aarch64 | 79.19 MB |
| musllinux x86_64 | 84.94 MB |
| musllinux aarch64 | 78.45 MB |

The manylinux/musllinux wheels — the ones affected by both the static-archive
bloat and the LTO amplifier — are 3–5× the size of the macOS wheels, which
carry the same static archive but neither the LTO IR bloat (macOS forces LTO
`FALSE`, above) nor §7.3's OpenBLAS duplication.

## 7.3 Root cause 2 — double OpenBLAS on manylinux

Cytnx links ARPACK unconditionally and by name only —
`find_library(ARPACK_LIB arpack REQUIRED)` (`external/Cytnx/CMakeLists.txt:307`,
already flagged in ch.02 §2.2 and ch.03 §3.1 point 2 as a version-blind,
name-only discovery mechanism with no ABI check). That same unversioned
discovery is the mechanical root of a second, independent size problem on
manylinux: the spot check in §7.1 found *exactly two* OpenBLAS shared objects
bundled into the wheel, `libopenblas-r0-...so` (serial) and
`libopenblasp-r0-...so` (threaded) — not one. #947 explains why both end up
required: cytnx's own extension links the **serial** OpenBLAS build directly,
while the manylinux CI image's prebuilt `arpack` package was itself linked
against the **threaded** OpenBLAS build; both are genuine `DT_NEEDED` entries
of the final extension, so `auditwheel repair` (ch.01 §1.2's default
cibuildwheel repair step, already discussed for a correctness purpose in
ch.03 §3.2/§3.3) vendors both rather than one redundant copy — this "isn't
`auditwheel` over-copying," it is two real, distinct runtime dependencies
(#947). #947 further confirms this affects both manylinux x86_64 and
aarch64, while musllinux is unaffected because its OpenBLAS packaging only
ever offers a single variant, so cytnx's own link and the musllinux `arpack`
package's link already agree.

The spot-checked sizes corroborate the scale of this specifically: the two
`.so` files together account for 22.2 MB packed (#947) — smaller than the
static-archive problem in §7.2, but a second, entirely independent source of
duplication with the same underlying cause (ARPACK's name-only,
version/variant-blind discovery, `CMakeLists.txt:307`, pulling in whatever
OpenBLAS variant the *ARPACK build* happens to have been linked against,
uncoordinated with the variant Cytnx's own CMake links).

## 7.4 The downstream-consumer tradeoff

**Why the headers and static archive exist at all.** This is not an
oversight with no purpose — `docs/source/install.rst` documents
`cytnx.__cpp_include__` and `cytnx.__cpp_lib__` as public, intended
Python-side accessors for the installed package's C++ header directory and
library directory respectively (`external/Cytnx/docs/source/install.rst:107–
108,112–113`), and shows them being read out and used directly to compile a
downstream C++ project against the installed `cytnx` (`export CYTNX_INC=...`
/ `export CYTNX_LIB=.../libcytnx.a`, `external/Cytnx/docs/source/
install.rst:141–142`, repeated as Makefile variables at `:172,174`). In other
words, the very thing bloating the Python wheel — the full header tree plus
`libcytnx.a` — is also the documented, deliberate mechanism by which a C++
user builds custom extensions against a `pip`-installed Cytnx. Any fix that
simply deletes these from the wheel breaks that documented workflow for
whoever currently relies on it (#947).

**How peer scientific-Python packages resolve the same tension.** #947
downloaded and inspected the same class of package directly rather than
relying on prior knowledge:

| Package | Static/linkable lib in wheel | Headers for downstream C/C++ builds? |
|---|---|---|
| **cytnx** 1.1.0 | Yes — `libcytnx.a`, 64.1 MB packed, the *entire* 180-object-file backend, duplicated in full alongside the compiled extension | Yes, via `cytnx.__cpp_include__`/`__cpp_lib__` (`docs/source/install.rst`) |
| **numpy** 2.5.0 | Yes, but tiny and narrowly scoped — `libnpymath.a` (53 KB) + `libnpyrandom.a` (71 KB): portable math/RNG helpers, not the computational core, which is compiled directly into the extension and never separately archived | Yes, via `numpy.get_include()`; the actual C API is a `PyCapsule`-backed function table (`import_array()`), needing no linking at all |
| **scipy** 1.18.0 | No | No general `get_include()`; only narrow `.pxd`/`.h` files for Cython `cimport` of specific submodules |
| **torch** 2.12.1 | No `.a`, but the shared-library equivalent — `torch/lib/` ships real, ordinarily-linkable `.so`s (`libc10.so`, `libtorch.so`, `libtorch_cpu.so`, ...) exporting genuine C++ symbols | Yes, via `torch.utils.cpp_extension`; officially supported `CppExtension`/`CUDAExtension` link against these |

(#947, table condensed from the issue's full comparison, which also covers
tensorflow/dask/xarray as non-comparable pure-Python or non-vendoring cases.)

#947's own conclusion from this comparison is the honest framing this
document adopts too: the ecosystem does **not** uniformly avoid shipping a
linkable artifact for downstream C++ builders — numpy and torch both do it
deliberately, for the same reason Cytnx's `install.rst` documents it. What
distinguishes Cytnx's case is not *that* it ships something linkable, but
*what* it ships: numpy scopes its archive to ~124 KB of narrow utility code
(not the computational core), and torch ships genuine shared libraries rather
than a raw static archive; Cytnx ships the entire backend, as a raw,
LTO-bloated static archive, with no narrower option (#947). scipy's
"ship nothing" approach is real but is not, on this evidence, the ecosystem
norm to target.

**The design space this leaves.** Three non-exclusive directions, named here
for §7.7 to size, not implemented in this chapter (§7.7):

1. **Exclude the archive/headers from the Python wheel; serve them
   separately** — e.g. a C++-only install artifact/tarball, or documentation
   pointing C++ consumers at `pip install .` / a source build instead of the
   PyPI wheel's Python-runtime install. Closest to scipy's approach; breaks
   the documented `__cpp_include__`/`__cpp_lib__` workflow for pip/conda users
   unless replaced.
2. **numpy-style narrow scoping** — identify and ship only the specific
   symbols/headers a downstream extension realistically needs, rather than
   the full 180-object backend. Requires design work to define what that
   narrow surface even is for Cytnx's API, unlike numpy's already-narrow
   math/RNG helpers.
3. **Split package** — a slim `cytnx` wheel (Python runtime only) plus an
   opt-in `cytnx-dev`/`cytnx-cpp`-style package or extra carrying headers +
   library, mirroring how MKL's `mkl` (runtime) and `mkl-devel` (headers +
   static, ch.02 §2.2) are already split on PyPI.

## 7.5 Build-matrix breadth

42 wheel files (#947) is itself a product, not a given: ch.01 §1.2's
published-artifact check on PyPI found `cytnx` 1.1.0 wheels for CPython
3.9–3.13, 3.14, and 3.14t (free-threaded) — **7 Python-version tags** — each
built in **6 platform variants** (`macosx_14_0_arm64`, `macosx_15_0_x86_64`,
manylinux x86_64, manylinux aarch64, musllinux x86_64, musllinux aarch64;
ch.01 §1.2). 7 × 6 = 42, matching #947's file count exactly — confirming the
42-wheel figure is fully explained by this Python-version × platform
multiplier, with no additional axis (e.g. no CUDA variant is published,
ch.01 §1.2) inflating it further.

This is a genuine second-order lever on top of §7.2–§7.3's per-wheel bloat:
total published bytes is (bytes per wheel) × (number of wheels), and this
chapter's other sections have only addressed the first factor. Two levers
exist on the second factor, named here as diagnosis only — not sized or
recommended, since evaluating them is fix work out of scope for this chapter
(§7.7): building against Python's **stable ABI (`abi3`)** would collapse the
CPython-version axis from 7 tags down to 1 per platform (a 7× reduction on
that axis alone, before touching per-wheel size at all) at the cost of
whatever pybind11/stable-ABI compatibility work that requires; alternatively,
**publishing fewer supported Python versions** (e.g. dropping 3.9 once it
reaches end-of-life, or not building the still-uncommon 3.14t
free-threaded variant) directly shrinks the same axis without an ABI-level
change. Both levers act on the *count* of wheels, independent of and
additive with §7.2's and §7.3's fixes to per-wheel *size*.

## 7.6 Our own recommendation's blind spot

Ch.06 §6.1 recommends a two-channel split: pip/wheels for the CPU
configuration, and **conda-forge as the primary channel for the GPU
configuration** (CUDA/cuTENSOR/cuQuantum), precisely because "the entire
proprietary GPU stack that pip/wheels cannot touch at all already exists as
conda-forge packages" (ch.06 §6.1). But ch.06 §6.3 Phase 3's roadmap does not
stop there — its last bullet names a secondary, optional path: "if a
pip/wheels GPU story is later wanted despite conda-forge being the primary
GPU home: adopt CuPy pattern C1 — per-CUDA-major-version wheels
(`cytnx-cuda12x`-style)" (ch.06 §6.3, citing ch.04 §4.2's C1 pattern), flagged
as "optional/secondary because §6.1 places GPU on conda-forge, not pip"
(ch.06 §6.3).

That optionality is exactly where this chapter's findings expose a blind
spot ch.06 never examined: **ch.06 never weighed the C1 per-CUDA-wheel option
against the storage-quota constraint this chapter establishes.** Run the
arithmetic ch.06 didn't: today's 42 CPU-only wheels already consume ~2.6 GB
of PyPI's 10 GB quota, with 3–4 releases of headroom left (#947, §7.1). A
`cytnx-cuda12x`/`cytnx-cuda13x`-style pip package, following CuPy's own C1
pattern (ch.04 §4.2), would add at minimum one new wheel per CUDA major
version per supported Linux platform/Python combination (CUDA is Linux-only
in Cytnx's own build graph, ch.02 §2.3(b); macOS has no CUDA at all,
ch.02 §2.2) — and, unless §7.2–§7.3's root causes are fixed first, each of
those new wheels would inherit the *same* static-archive-plus-LTO bloat this
chapter measured on the existing manylinux CPU wheels (§7.2), on top of
whatever CUDA runtime libraries a self-contained GPU wheel would additionally
need to bundle. A handful of new, CUDA-major-version-suffixed wheels, each
built from a codebase that has not fixed the ~66–70%-static-archive problem,
would burn through the remaining 3–4-release quota headroom considerably
faster than the current CPU-only cadence does.

Put plainly: **ch.06's own §6.1 decision to make conda-forge, not pip, the
GPU channel already protects the project from the scenario this chapter
quantifies** — conda-forge carries no analogous per-project storage quota in
anything this document has established — but ch.06 arrived at that
conclusion for reasons entirely unrelated to wheel size (robustness,
dependency coverage, the proprietary-license/CUDA-coupling story of ch.02
§2.3(b)); §6.1's text never mentions storage quota at all. The blind spot is
not that ch.06's recommendation is wrong — on this evidence it is
directionally *right*, and for an additional reason it didn't know it had —
it is that ch.06's own Phase 3 roadmap leaves the C1 pip-wheel path open as
"optional/secondary" without ever flagging that pursuing it *before* fixing
§7.2's/§7.3's root causes would be actively dangerous to the project's PyPI
quota headroom. Any future decision to exercise that optional C1 path should
treat §7.7's fixes below as a hard prerequisite, not a parallel nice-to-have.

## 7.7 Fixes → roadmap

The three items below are the concrete fix work this chapter's diagnosis
points to. Consistent with this document's role throughout (ch.06 §6.4:
"this repo's job was to diagnose... not... build and ship the artifacts"),
none of it is implemented here — this section only names and sizes the work.
Each is phrased so it could be pasted into a GitHub issue, matching ch.06
§6.2's format:

1. **Stop installing the `ARCHIVE`/headers/CMake-config outputs into the
   Python wheel.** Gate `CMakeLists.txt:457–467`'s `ARCHIVE`/`PUBLIC_HEADER`
   install and `:482–485`'s header-tree install behind a CMake install
   `COMPONENT` that scikit-build-core's `install.components` config is set to
   skip when building the Python wheel, while keeping them for a C++-only
   `cmake --install`. This is #947's own primary suggested direction, with the
   explicit caveat that it drops the documented `__cpp_include__`/
   `__cpp_lib__` workflow (§7.4) unless one of §7.4's three design-space
   options replaces it. **Effort: S–M** (CMake `COMPONENT` tagging plus a
   `scikit-build-core` config change; the harder part is deciding which of
   §7.4's three replacement options to pursue, if any, for the C++-consumer
   workflow). **Acceptance signal:** the manylinux x86_64 wheel shrinks from
   ~92.54 MB to roughly ~30 MB packed, matching #947's own estimate for this
   fix alone, with proportional drops on the other five platform variants.

2. **Fix the LTO bloat.** Either force-disable
   `CMAKE_INTERPROCEDURAL_OPTIMIZATION` for the archived `cytnx` static target
   the way `CMakeLists.txt:223–231` already does on Apple, or run a proper
   LTO reduction/link step before the object code is placed into the
   installed archive, so the shipped `.o`s hold compiled machine code rather
   than raw GIMPLE bytecode. **Effort: M** (requires understanding why LTO
   was enabled on Linux in the first place — parallel-compile speed per the
   comment at `CMakeLists.txt:223–224` — and whether disabling it, or adding
   a reduction pass, regresses that). **Acceptance signal:** extracting the
   same `Svd.cpp.o` object from a rebuilt Linux archive shows a populated
   `.text` section of a size comparable to the macOS build's 199 KB, not
   #947's measured 880 KB of pure IR.

3. **Dedupe OpenBLAS on manylinux by aligning ARPACK and Cytnx to one
   variant.** Either build/select an `arpack` for CI that links the same
   OpenBLAS variant (serial) `find_library(ARPACK_LIB arpack REQUIRED)`
   (`CMakeLists.txt:307`) already causes Cytnx itself to link, or switch
   Cytnx's own link to match whichever variant the manylinux image's `arpack`
   package uses — either direction collapses two `DT_NEEDED` OpenBLAS copies
   into one. musllinux already demonstrates the target state, since its
   single-variant OpenBLAS packaging means this duplication never occurs
   there (§7.3, #947). **Effort: S–M** (a CI/build-script change to pin or
   select the ARPACK build, not a Cytnx source change). **Acceptance signal:**
   `unzip -v` on a rebuilt manylinux wheel shows exactly one
   `libopenblas*.so`, not two, shrinking the wheel by roughly the 22.2 MB
   the duplicate copy currently costs (#947).

These three items are a natural **Phase 0** ahead of ch.06 §6.3's existing
Phase 1–3 roadmap: they are wheel-hygiene fixes orthogonal to ch.06's
discovery/RPATH/CI-coverage concerns, but §7.6 already established why they
are a *prerequisite*, not merely a nice-to-have, before ch.06's Phase 3 GPU
work ever exercises the optional pip-wheel C1 path. Actually implementing,
testing, and landing these three fixes as an upstream pull request against
`Cytnx-dev/Cytnx` is a separate **follow-up** project and is **out of scope**
for this analysis repository, for the same reasons ch.06 §6.4 gives for its
own roadmap: this repo holds `external/Cytnx` as a read-only submodule with
no write access to the upstream build files, and validating any of these
three fixes requires actually rebuilding and re-measuring wheels against the
real `Cytnx-dev/Cytnx` CI — work that belongs in the Cytnx source tree, not
in this design-analysis repo.
