# Wheel-Artifact Efficiency Chapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new chapter `docs/07-wheel-artifact-efficiency.md` on the Cytnx PyPI wheel size/quota problem (issue #947) and make surgical, cross-referenced edits to ch.02, ch.06, ch.00, and the README so the analysis document stays authoritative and internally consistent.

**Architecture:** One new content chapter written in the document's established layered, heavily-cited style, backed by (a) #947 as the cited source investigation for size measurements, (b) independent verification of every causal/CMake claim against the pinned Cytnx v1.1.0 submodule, and (c) a one-wheel spot check. Then additive notes to the dependency taxonomy (ch.02), a new roadmap Phase 0 (ch.06), a one-sentence executive-summary flag (ch.00), and a README link — no restructuring of existing chapters.

**Tech Stack:** Markdown (GitHub-flavored), `git`/`grep`/`gh`/`curl`/`unzip`/`python3` for evidence gathering and verification. Working repo: `/Users/pcchen/github/pcchen/cytnx_design_deployment`, branch `build-analysis-doc`.

## Global Constraints

- Reference point: Cytnx **v1.1.0** (`8d96d928d22cc176615e97cafdb7a3cef66cb732`), the read-only submodule at `external/Cytnx`. Never modify the submodule.
- Every claim about Cytnx's *current* behavior cites a specific `external/Cytnx/…:NN` file:line (open the file; never invent line numbers).
- Issue **#947** (`https://github.com/Cytnx-dev/Cytnx/issues/947`) is cited as the source investigation for all size *measurements* and the numpy/scipy/torch comparison — attributed, not silently absorbed. Fetch exact figures from it (`gh issue view 947 --repo Cytnx-dev/Cytnx`); do not hardcode remembered numbers.
- Perform a **single `cp312` manylinux x86_64 wheel** spot check to corroborate the two headline numbers (`libcytnx.a` ≈ its share of packed size; two OpenBLAS `.so`s). If the download fails after a genuine retry, state that plainly in the chapter and lean on #947 — never fabricate figures.
- **Analysis only.** Do NOT implement any fix, patch `pyproject.toml`/CMake, or open a PR. Chapter 07 §7.7 names the fix work as a separate follow-up, out of scope.
- No placeholder text (`TODO`/`TBD`/`FIXME`) in any committed deliverable.
- ch.02 edits are **additive notes only** — no restructuring of existing profiles.
- Windows stays noted-where-it-differs (ships no wheels today).
- Whole-document invariants after this work: 8 chapters `docs/00`–`07` exist, no placeholders in deliverables, all internal `.md` links resolve.
- Commit only the files each task names; never `git add` the `.superpowers/` scratch dir.

## File Structure

| Path | Responsibility | Task |
|------|----------------|------|
| `docs/07-wheel-artifact-efficiency.md` | New chapter: size/quota problem, two root causes, tradeoff, matrix breadth, blind spot, fixes | 1 |
| `docs/02-dependency-taxonomy.md` | Add 3 additive size-footprint notes (OpenBLAS, ARPACK, `libcytnx.a`), each → ch.07 | 2 |
| `docs/06-recommended-strategy.md` | Insert roadmap Phase 0 (quota triage); add C++-SDK-exclusion to §6.2 | 3 |
| `docs/00-executive-summary.md` | One sentence flagging the quota wall → ch.07 | 4 |
| `README.md` | Add chapter 07 to "How to read this" | 4 |

**Task dependencies:** Task 1 first (Tasks 2–4 cross-reference ch.07's sections). Tasks 2, 3, 4 each depend on Task 1; Task 4 runs last (its whole-document link check must see ch.07 and the new README link).

---

### Task 1: Chapter 07 — wheel-artifact efficiency

**Files:**
- Create: `docs/07-wheel-artifact-efficiency.md`

**Interfaces:**
- Consumes: the pinned submodule `external/Cytnx/`; issue #947; one downloaded wheel.
- Produces: sections `7.1`–`7.7` that Tasks 2–4 cross-reference by number (e.g. "ch.07 §7.2", "§7.3", "§7.7").

- [ ] **Step 1: Acceptance check (expect fail)**

Run: `test -f docs/07-wheel-artifact-efficiency.md && echo OK || echo MISSING`
Expected: `MISSING`

- [ ] **Step 2: Pull the exact figures from issue #947**

```bash
gh issue view 947 --repo Cytnx-dev/Cytnx --json title,body | python3 -c "import sys,json;d=json.load(sys.stdin);print(d['title']);print(d['body'])" | head -120
```
Record verbatim, for citation: total wheel size and count (~2.6 GB / 42 wheels), the PyPI quota (10 GB) and headroom (~3–4 releases), the per-wheel size table, the `libcytnx.a` packed size and its ~70% share on manylinux x86_64, the `Svd.cpp.o` 880 KB-IR-vs-199 KB-machine figure, the two OpenBLAS `.so` names/sizes, and the numpy/scipy/torch comparison rows.

- [ ] **Step 3: Verify the causal/CMake claims against the submodule**

Open the real files and record exact line numbers:
```bash
grep -n -E "add_library\(cytnx STATIC|install\(|ARCHIVE|EXPORT|PUBLIC_HEADER|CMAKE_INTERPROCEDURAL_OPTIMIZATION|IF *\(APPLE" external/Cytnx/CMakeLists.txt
grep -n -E "find_library\(ARPACK|arpack" external/Cytnx/CMakeLists.txt
grep -n -E "__cpp_include__|__cpp_lib__|cpp_include|cpp_lib" external/Cytnx/docs/source/install.rst
```
Confirm: the `cytnx` STATIC target and the `install(... ARCHIVE ...)`/header/CMake-config install that places `libcytnx.a`+headers into the wheel path; the LTO block (`IF (APPLE) ... FALSE ELSE ... TRUE`, expected `CMakeLists.txt:223-231` — verify) driving slim-LTO on Linux; the ARPACK link line; and that `install.rst` documents `cytnx.__cpp_include__`/`__cpp_lib__` as the intended C++-consumer surface. Record each `file:line`.

- [ ] **Step 4: Spot-check one wheel (corroborate the two headline numbers)**

```bash
SP="/private/tmp/claude-501/-Users-pcchen-github-pcchen-cytnx-design-deployment/36981133-c51d-4977-a4f9-eaa5fc35a6df/scratchpad"
mkdir -p "$SP/wheel" && cd "$SP/wheel"
# Find a cp312 manylinux x86_64 wheel URL from PyPI and download it
URL=$(curl -fsSL https://pypi.org/pypi/cytnx/1.1.0/json | python3 -c "import sys,json;d=json.load(sys.stdin);print(next(u['url'] for u in d['urls'] if 'cp312' in u['filename'] and 'manylinux' in u['filename'] and 'x86_64' in u['filename']))")
echo "URL=$URL"; curl -fsSL -o w.whl "$URL" && ls -la w.whl
# Per-file compressed (packed) sizes; find libcytnx.a share and count OpenBLAS
unzip -v w.whl | grep -E "libcytnx\.a|libopenblas|openblas" 
python3 - <<'PY'
import zipfile
z=zipfile.ZipFile("w.whl")
infos=z.infolist()
total=sum(i.compress_size for i in infos)
a=[i for i in infos if i.filename.endswith("libcytnx.a")]
ob=[i for i in infos if "openblas" in i.filename.lower() and i.filename.endswith((".so",)) or ("openblas" in i.filename.lower() and ".so" in i.filename.lower())]
print("total packed bytes:", total)
for i in a: print("libcytnx.a packed:", i.compress_size, "=> %.0f%% of packed" % (100*i.compress_size/total))
print("openblas .so count:", len(ob), [i.filename for i in ob])
PY
cd /Users/pcchen/github/pcchen/cytnx_design_deployment
```
Record: the measured `libcytnx.a` packed share (%), and the OpenBLAS `.so` count (expect 2) with filenames. Do NOT commit the wheel (it lives in scratchpad). If any network step fails after a retry, note "spot check could not run: <reason>" and proceed citing #947 only.

- [ ] **Step 5: Write the chapter**

Write `docs/07-wheel-artifact-efficiency.md` with these sections. Every current-behavior claim carries an `(external/Cytnx/…:NN)` citation; every size measurement carries a `(#947)` attribution; spot-check numbers are marked as independently measured.
```markdown
# 07 — Wheel-Artifact Efficiency

Reference commit: Cytnx `v1.1.0` (`8d96d928d22cc176615e97cafdb7a3cef66cb732`).
Size measurements are attributed to [Cytnx-dev/Cytnx#947](https://github.com/Cytnx-dev/Cytnx/issues/947);
causal/CMake claims are verified against the pinned submodule; the §7.1 spot
check was performed independently for this chapter.

## 7.1 The problem — a different axis
- PyPI storage quota (10 GB) vs current footprint (~2.6 GB / 42 wheels), ~3–4
  releases of headroom (#947). Contrast with ch.03's *correctness* axis.
- One-wheel spot-check result (or a plain statement it could not run).

## 7.2 Root cause 1 — the C++ SDK shipped inside the Python wheel
- What is packaged and why (cytnx STATIC target install tree → scikit-build-core;
  cite install lines). LTO amplifier (cite the LTO block; #947's Svd.cpp.o IR
  figure). Spot-check: libcytnx.a ≈ N% of packed size.

## 7.3 Root cause 2 — double OpenBLAS on manylinux
- serial (cytnx) + threaded (manylinux arpack), both DT_NEEDED → auditwheel
  vendors both; musllinux unaffected. Spot-check: two .so present. Cite the
  ARPACK link line; attribute the variant split to #947.

## 7.4 The downstream-consumer tradeoff
- Headers + .a shipped deliberately (`cytnx.__cpp_include__`/`__cpp_lib__`,
  cite install.rst). Peer approaches: numpy (tiny scoped .a + PyCapsule),
  scipy (none), torch (shared libs) (#947 comparison). Design space (exclude +
  serve separately / numpy-style scoping / split package).

## 7.5 Build-matrix breadth
- 42 wheels ≈ platforms × Python versions; the multiplier and its second-order
  levers (stable-ABI/abi3, fewer Python versions). Diagnosis only.

## 7.6 Our own recommendation's blind spot
- per-CUDA wheels (ch.04 C1 / ch.06) × bloated wheels would hit the quota fast;
  size fixes are a prerequisite; conda-forge for GPU (ch.06) also protects it.

## 7.7 Fixes → roadmap
- (1) exclude ARCHIVE/headers/CMake-config from the wheel; (2) fix LTO bloat;
  (3) dedupe OpenBLAS by aligning arpack+cytnx to one variant. Each: rough
  effort + acceptance signal. These feed ch.06 Phase 0. The upstream-PR fix
  project is named as a separate follow-up, out of scope for this repo.
```

- [ ] **Step 6: Verify coverage, citations, scope discipline**

```bash
for S in 7.1 7.2 7.3 7.4 7.5 7.6 7.7; do grep -q "## $S" docs/07-wheel-artifact-efficiency.md && echo "OK $S" || echo "MISSING $S"; done
grep -c "external/Cytnx/" docs/07-wheel-artifact-efficiency.md          # expect >= 4 (causal claims cited)
grep -c "947" docs/07-wheel-artifact-efficiency.md                        # expect >= 3 (#947 attributions)
grep -qiE "numpy|scipy|torch" docs/07-wheel-artifact-efficiency.md && echo "peer comparison present"
grep -qiE "follow-up|out of scope" docs/07-wheel-artifact-efficiency.md && echo "fix work scoped out"
grep -nE "TODO|TBD|FIXME" docs/07-wheel-artifact-efficiency.md || echo "no placeholders"
```
Expected: `OK` for all 7 sections; citation count ≥ 4; `947` count ≥ 3; peer comparison present; fix work scoped out; `no placeholders`.

- [ ] **Step 7: Commit**

```bash
git add docs/07-wheel-artifact-efficiency.md
git commit -m "docs: add chapter 07 wheel-artifact efficiency"
```

---

### Task 2: ch.02 dependency-taxonomy size notes

**Files:**
- Modify: `docs/02-dependency-taxonomy.md`

**Interfaces:**
- Consumes: ch.07 §7.2 and §7.3 (created in Task 1).
- Produces: three additive cross-reference notes; no new consumable interface.

- [ ] **Step 1: Acceptance check (expect fail)**

Run: `grep -c "ch.07\|07-wheel-artifact" docs/02-dependency-taxonomy.md`
Expected: `0`

- [ ] **Step 2: Locate the three insertion points**

```bash
grep -n -E "^### OpenBLAS|^### ARPACK|libcytnx\.a" docs/02-dependency-taxonomy.md
```
Record the OpenBLAS profile heading, the ARPACK profile heading, and the existing `libcytnx.a`/hptt sentence (from ch.02 §2.2, "Because it is statically linked in, hptt has no independent runtime footprint … `libcytnx.a`/`pycytnx*.so`").

- [ ] **Step 3: Add the three additive notes**

Append one sentence to each of the three locations (do not restructure surrounding prose). Exact text to add:
- To the **OpenBLAS** profile: `**Size footprint:** on manylinux the repaired wheel carries *two* OpenBLAS copies (a serial and a threaded build), inflating the redistribution footprint — see ch.07 §7.3.`
- To the **ARPACK** profile: `**Size footprint:** the manylinux prebuilt \`arpack\` links the *threaded* OpenBLAS while cytnx links the serial one, which is what forces the second bundled copy — see ch.07 §7.3.`
- To the **`libcytnx.a`/hptt** sentence: append `Note that \`libcytnx.a\` itself is packaged into the Python wheel, where ch.07 §7.2 identifies it as the single largest size cost (dead weight at Python runtime).`

- [ ] **Step 4: Verify notes present, additive, and libraries still all covered**

```bash
grep -c "ch.07 §7.3" docs/02-dependency-taxonomy.md      # expect 2 (OpenBLAS + ARPACK)
grep -c "ch.07 §7.2" docs/02-dependency-taxonomy.md      # expect 1 (libcytnx.a note)
for L in hptt OpenBLAS MKL "CUDA toolkit" cuTENSOR cuQuantum ARPACK pybind11 Boost; do grep -q "$L" docs/02-dependency-taxonomy.md && echo "OK $L" || echo "MISSING $L"; done
grep -nE "TODO|TBD|FIXME" docs/02-dependency-taxonomy.md || echo "no placeholders"
```
Expected: `ch.07 §7.3` count = 2, `ch.07 §7.2` count = 1, `OK` for all 9 libraries, `no placeholders`.

- [ ] **Step 5: Commit**

```bash
git add docs/02-dependency-taxonomy.md
git commit -m "docs: cross-reference wheel-size footprint from ch.02 taxonomy"
```

---

### Task 3: ch.06 roadmap Phase 0 + §6.2 update

**Files:**
- Modify: `docs/06-recommended-strategy.md`

**Interfaces:**
- Consumes: ch.07 §7.7 (the three fixes, created in Task 1).
- Produces: a `Phase 0` in the §6.3 roadmap and a fourth §6.2 highest-impact change, both referenceable by later readers.

- [ ] **Step 1: Acceptance check (expect fail)**

Run: `grep -c "Phase 0" docs/06-recommended-strategy.md`
Expected: `0`

- [ ] **Step 2: Locate the §6.2 change list and the §6.3 phase list**

```bash
grep -n -E "## 6.2|## 6.3|Phase 1|highest-impact|### 6" docs/06-recommended-strategy.md
```
Record where §6.2's numbered changes are and where §6.3's `Phase 1` begins.

- [ ] **Step 3: Insert Phase 0 ahead of the existing phases (§6.3)**

Immediately before the existing `Phase 1`, insert:
```markdown
- **Phase 0 — immediate quota triage (most time-sensitive).** PyPI's 10 GB
  per-project quota leaves ~3–4 releases of headroom at the current ~2.6 GB /
  release (ch.07 §7.1, #947), so this precedes the correctness/GPU phases below.
  1. Exclude the C++ SDK (`libcytnx.a` + headers + CMake config) from the Python
     wheel via a scikit-build-core install-component exclusion (ch.07 §7.2, §7.7).
     *Effort: S–M. Acceptance: wheel no longer contains `lib*/libcytnx.a` or
     `include/`; per-release footprint drops sharply.*
  2. Fix the Linux slim-LTO bloat so the shipped archive is machine code, not
     unreduced GIMPLE IR — only relevant if the `.a` is still shipped (ch.07 §7.2).
     *Effort: S. Acceptance: extracted object files show real `.text`.*
  3. Dedupe OpenBLAS by aligning the manylinux `arpack` and cytnx builds to one
     variant (ch.07 §7.3, §7.7). *Effort: M. Acceptance: repaired manylinux wheel
     bundles a single OpenBLAS `.so`.*
  Building/validating these configs is the separate follow-up fix project
  (ch.07 §7.7), out of scope for this analysis.
```
(If the existing phases are written as `### Phase 1` headings rather than list items, match that heading style for `### Phase 0` and keep the numbered fixes as its body.)

- [ ] **Step 4: Add the C++-SDK-exclusion change to §6.2**

Append to §6.2's numbered highest-impact changes a new item:
```markdown
N. **Stop shipping the C++ SDK inside the Python wheel** (ch.07 §7.2) — the
   single most deadline-driven change, because PyPI's storage quota is the
   nearest hard wall (#947); pairs with the LTO-bloat and double-OpenBLAS fixes
   (ch.07 §7.3). *Effort: S–M.*
```
(Use the next sequential number for `N`.)

- [ ] **Step 5: Verify Phase 0, §6.2 item, and scope discipline**

```bash
grep -qi "Phase 0" docs/06-recommended-strategy.md && echo "Phase 0 present"
grep -qi "immediate quota triage" docs/06-recommended-strategy.md && echo "triage framing present"
grep -qi "C++ SDK\|libcytnx.a" docs/06-recommended-strategy.md && echo "6.2 change present"
grep -qiE "follow-up|out of scope" docs/06-recommended-strategy.md && echo "fix work still scoped out"
grep -nE "TODO|TBD|FIXME" docs/06-recommended-strategy.md || echo "no placeholders"
```
Expected: all four `present`/scoped-out lines, `no placeholders`.

- [ ] **Step 6: Commit**

```bash
git add docs/06-recommended-strategy.md
git commit -m "docs: add roadmap Phase 0 (quota triage) to ch.06"
```

---

### Task 4: ch.00 flag + README link + whole-document verification

**Files:**
- Modify: `docs/00-executive-summary.md`, `README.md`

**Interfaces:**
- Consumes: ch.07 (Task 1), and the edits from Tasks 2–3.
- Produces: the final consistent, link-clean 8-chapter document.

- [ ] **Step 1: Acceptance check (expect fail)**

Run: `grep -c "07-wheel-artifact-efficiency" docs/00-executive-summary.md README.md`
Expected: `0`

- [ ] **Step 2: Add the executive-summary sentence**

In `docs/00-executive-summary.md`, add one sentence (in the problem or highest-impact area) with a relative sibling link:
```markdown
Separately and most urgently, Cytnx's PyPI wheels are ~2.6 GB per release against a 10 GB storage quota — roughly 3–4 releases from a hard wall ([wheel-artifact efficiency](07-wheel-artifact-efficiency.md)).
```

- [ ] **Step 3: Add the README chapter link**

In `README.md`, add to the "How to read this" numbered list, after item 7:
```markdown
8. [Wheel-artifact efficiency](docs/07-wheel-artifact-efficiency.md)
```

- [ ] **Step 4: Whole-document verification**

```bash
ls docs/0{0,1,2,3,4,5,6,7}-*.md | wc -l            # expect 8
! grep -rn -E "TODO|TBD|FIXME" docs/0*.md README.md && echo "no placeholders"
python3 - <<'PY'
import re, pathlib, sys
bad=[]
for md in list(pathlib.Path("docs").glob("*.md"))+[pathlib.Path("README.md")]:
    for m in re.finditer(r"\]\((?!https?:)([^)#]+\.md)", md.read_text()):
        t=(md.parent/m.group(1)).resolve()
        if not t.exists(): bad.append((str(md), m.group(1)))
print("broken links:", bad or "none")
sys.exit(1 if bad else 0)
PY
```
Expected: chapter count `8`, `no placeholders`, `broken links: none`.

- [ ] **Step 5: Commit**

```bash
git add docs/00-executive-summary.md README.md
git commit -m "docs: flag wheel-size quota in summary and link chapter 07"
```

---

## Self-Review

**Spec coverage** (each spec requirement → task):
- New chapter 07 with sections 7.1–7.7 → Task 1. ✅
- #947 cited for measurements; CMake claims verified against submodule; one-wheel spot check → Task 1 Steps 2–4 + verification. ✅
- Downstream-consumer tradeoff / numpy-scipy-torch (§7.4), matrix breadth (§7.5), per-CUDA blind spot (§7.6) → Task 1 §7.4–7.6. ✅
- ch.02 three additive notes → Task 2. ✅
- ch.06 Phase 0 + §6.2 change → Task 3. ✅
- ch.00 one sentence → Task 4 Step 2. ✅
- README chapter link → Task 4 Step 3. ✅
- Whole-document invariants (8 chapters, no placeholders, links resolve) → Task 4 Step 4. ✅
- Analysis-only / fixes out of scope → enforced in Global Constraints and verified in Task 1 Step 6 / Task 3 Step 5. ✅

**Placeholder scan:** the `TODO|TBD|FIXME` tokens in verification `grep` commands check the *deliverables* are placeholder-free; they are not plan placeholders. No unfilled plan steps.

**Type/name consistency:** section identifiers (`§7.2`, `§7.3`, `§7.7`), the reference commit hash, the file paths, the "Phase 0 — immediate quota triage" label, and the 9-library list are used identically across Tasks 1–4 and the cross-reference greps.
