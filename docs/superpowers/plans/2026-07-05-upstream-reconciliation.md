# Upstream Reconciliation & Status Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reconcile the ch.06/07 deployment roadmap against the in-flight upstream `Cytnx-dev/Cytnx` issue/PR cluster, publish an umbrella tracking issue, and name the first genuine-gap PR — producing no code PR in this sub-project.

**Architecture:** Two artifacts plus one named output. (1) A reconciliation map committed to *this* analysis repo (`docs/08-upstream-reconciliation.md`) that classifies every roadmap item against real upstream state, verified against a `master` clone (not the read-only v1.1.0 submodule). (2) An umbrella "status board" issue filed on upstream that references — never duplicates — the existing cluster. (3) A first-PR candidate section inside the map, chosen by three gates.

**Tech Stack:** Markdown (GitHub-flavored), `gh` CLI (issue/PR queries + issue creation; token has `repo`+`workflow`), `git` (a `master` clone for code-state verification), `grep`.

## Global Constraints

- **Verification source is the `master` clone**, never `external/Cytnx` (v1.1.0). v1.1.0 line numbers are stale and must be re-checked against `master` before any "verified" claim.
- **No code PR** is written in this sub-project. The first PR is *identified*, not implemented.
- **Reference, never duplicate** the upstream cluster in the tracking issue (#945–#949, #962–#969, PRs #950/#967).
- **The upstream issue must be self-contained** — this analysis repo is private, so the issue cannot link to `docs/*` here.
- **Filing the upstream issue is outward-facing and hard to reverse** — the issue body must be approved by the user before `gh issue create` runs.
- **Platform ceiling (macOS/darwin):** record per row whether an item is buildable/verifiable locally. Linux+CUDA-only repros (e.g. #948) are not.
- **First-PR gates (all three required):** classification `gap` · macOS-verifiable · not `gap-decision-coupled` on #969.
- Target repo default branch is **`master`**; author has `admin`/`push` (no fork needed).

---

### Task 1: Set up the `master` verification clone

**Files:**
- Create: `../Cytnx-upstream/` (a git clone; sibling of this repo, not committed here)

**Interfaces:**
- Consumes: nothing.
- Produces: a clean `Cytnx-dev/Cytnx` checkout at `master` HEAD, referenced by later tasks as `../Cytnx-upstream`.

- [ ] **Step 1: Write the acceptance check and run it (expect fail)**

Run:
```bash
git -C ../Cytnx-upstream rev-parse --abbrev-ref HEAD 2>&1 || echo MISSING
```
Expected: `MISSING` (clone does not exist yet).

- [ ] **Step 2: Clone upstream at `master`**

Run:
```bash
gh repo clone Cytnx-dev/Cytnx ../Cytnx-upstream -- --branch master --depth 1
```
Expected: clone completes; `../Cytnx-upstream/CMakeLists.txt` exists.

- [ ] **Step 3: Verify the clone is on `master` and record its HEAD**

Run:
```bash
git -C ../Cytnx-upstream rev-parse --abbrev-ref HEAD
git -C ../Cytnx-upstream log --oneline -1
```
Expected: prints `master` and the current HEAD commit. Note the HEAD short-SHA — it is recorded in the map header in Task 2 (so "verified on master" is reproducible).

- [ ] **Step 4: Confirm the submodule is NOT the verification source**

Run:
```bash
git -C external/Cytnx describe --tags 2>&1 | head -1
```
Expected: shows `v1.1.0` (or a v1.1.0-based describe) — confirming the submodule is the stale reference, distinct from `../Cytnx-upstream`. No commit in this task (the clone is external and uncommitted).

---

### Task 2: Build the reconciliation map

**Files:**
- Create: `docs/08-upstream-reconciliation.md`

**Interfaces:**
- Consumes: the `../Cytnx-upstream` clone (Task 1) and its recorded HEAD SHA.
- Produces: `docs/08-upstream-reconciliation.md` containing (a) a header noting the clone HEAD + date, (b) one reconciliation row per item in the inventory below, every "Verified on `master`?" cell filled, (c) a "First-PR candidate" section naming exactly one item that passes all three gates.

**Item inventory to reconcile** (source-of-truth list; each becomes one row):

| # | Item (label) | Our ref | Candidate upstream issue/PR |
|---|---|---|---|
| 0.1 | Exclude C++ SDK (`libcytnx.a`+headers+CMake config) from Python wheel | ch.07 §7.2, §7.7 | PR #967, issue #947 |
| 0.2 | Fix Linux slim-LTO `.a` GIMPLE-IR bloat | ch.07 §7.2 | (obviated if #967 stops shipping `.a`?) |
| 0.3 | Dedup OpenBLAS across manylinux arpack+cytnx builds | ch.07 §7.3, §7.7 | issue #947 |
| 1.1 | `*_FOUND TRUE` dead-code bug in FindCUTENSOR/FindCUQUANTUM | ch.03 §3.1, ch.02 §2.2 | PR #950, issues #945/#946 |
| 1.2 | Add `$ORIGIN` / remove unexpanded `~` in INSTALL_RPATH | ch.03 §3.2 | (check if dissolved on master; see #948) |
| 1.3 | Remove stale legacy `CUDA_*_LIBRARY`/`linkflags.tmp` dead block | ch.02 §2.2 | issue #949, PR #950 |
| 1.4 | Correct stale `adv_install.rst`: `USE_OMP`, install-prefix, missing `USE_CUTENSOR`/`USE_CUQUANTUM`/`BACKEND_TORCH`, macOS OpenBLAS-vs-MKL | ch.01 §1.4, ch.02 §2.2 | docs edits in PR #950 & #967 (contested) |
| 1.5 | (new, from brainstorm) installed CUDA extension not rpath-pinned to build toolkit | — | issue #948 |
| 2.1 | Document `pip install cytnx` / `uv add cytnx` as first-class in `install.rst` | ch.01 §1.3–1.4, ch.05 §5.4 | — |
| 2.2 | SciPy S1/S2: pinned OpenBLAS/ARPACK build-only source + documented auditwheel/delocate | ch.04 §4.1–4.3 | — |
| 2.3 | Move conda from `kaihsinwu` to conda-forge feedstock; fix version lag | ch.01 §1.2/§1.4 | — |
| 2.4 | macOS Accelerate-linked BLAS option (optional) | ch.05 §5.3, ch.04 §4.1 | — |
| 3.1 | CI workflow building/importing `openblas-cuda` on a CUDA runner | §6.2 item 3, ch.01 §1.1–1.2 | — |
| 3.2 | CuPy C2 ordered search chain in FindCUTENSOR/FindCUQUANTUM | ch.03 §3.1, ch.04 §4.2 | issue #946, PR #950 |
| 3.3 | conda-forge GPU deps (cudatoolkit/cutensor/cuquantum) — CuPy C3 | ch.02 §2.1, ch.04 §4.2 | — |
| 3.4 | Per-CUDA-major wheels (`cytnx-cuda12x`) — CuPy C1 (optional) | ch.04 §4.2 | #969 |
| 3.5 | Reconcile advertised-but-unbuilt `cytnx_cuda` conda package | ch.01 §1.4 | — |
| S | Packaging-model strategy (PyTorch vs JAX variants) | ch.04–06 | issue #969 (+#966) |

- [ ] **Step 1: Write the acceptance check and run it (expect fail)**

Run:
```bash
test -f docs/08-upstream-reconciliation.md && echo EXISTS || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Create the map skeleton with header and the full row table**

Create `docs/08-upstream-reconciliation.md` with this header and a table containing every inventory row (columns: Item · Our ref · Upstream issue/PR · Status · Verified on `master`? · macOS-verifiable? · Classification). Leave `Status`, `Verified`, `macOS-verifiable`, `Classification` blank for now:

```markdown
# 08 — Upstream Reconciliation & Status Board

**Reconciled against:** `Cytnx-dev/Cytnx` `master` @ `<HEAD-short-sha from Task 1>` on 2026-07-05.

Maps every ch.06 (Phases 0–3) and ch.07 roadmap item to its current upstream
reality. Classifications: `covered` (an open/merged PR already does it) ·
`dissolved` (the v1.1.0 finding no longer holds on `master`) · `gap` (real,
actionable now) · `gap-decision-coupled` (real gap whose fix depends on the
#969 packaging-model decision) · `superseded` (subsumed/reframed by a stronger
upstream artifact, chiefly #969).

| Item | Our ref | Upstream | Status | Verified on `master`? | macOS-verifiable? | Classification |
|------|---------|----------|--------|-----------------------|-------------------|----------------|
| 0.1 Exclude C++ SDK from wheel | ch.07 §7.2 | #967, #947 | | | | |
| ... (one row per inventory item, including S) | | | | | | |
```

- [ ] **Step 3: Fill the `Status` column from `gh`**

For each row, query upstream and set `Status` to `merged`/`open-PR`/`open-issue`/`none`:
```bash
for n in 967 947 950 945 946 949 948 969 966 962 963; do
  echo "#$n $(gh issue view $n -R Cytnx-dev/Cytnx --json state,title --jq '.state+"  "+.title' 2>/dev/null || gh pr view $n -R Cytnx-dev/Cytnx --json state,title --jq '"PR "+.state+"  "+.title')"
done
```
Expected: prints state+title per number; transcribe into each row's `Status` cell.

- [ ] **Step 4: Fill `Verified on master?` by diffing each cited file against the clone**

For every row, confirm the *current* code state in `../Cytnx-upstream`. Worked examples (run the analogous check per row):

```bash
# 0.1 SDK exclusion — is the COMPONENT/install.components change already on master, or only in PR #967?
git -C ../Cytnx-upstream grep -nE "install\.components|COMPONENT (libraries|headers|Development)" pyproject.toml CMakeLists.txt

# 1.2 $ORIGIN / '~' — confirm the v1.1.0 '~' finding is dissolved on master
git -C ../Cytnx-upstream grep -nE "INSTALL_RPATH|@loader_path|~" CMakeLists.txt

# 1.1 / 1.3 CUDA finders & legacy block — is the fix on master or still only in open PR #950?
git -C ../Cytnx-upstream grep -nE "_FOUND|CUDA_.*_LIBRARY|linkflags\.tmp" cmake/Modules/FindCUTENSOR.cmake CytnxBKNDCMakeLists.cmake

# 1.4 docs — what does master's adv_install.rst currently say?
git -C ../Cytnx-upstream grep -nE "USE_OMP|USE_HPTT|/usr/local/cytnx|USE_CUTENSOR|USE_CUQUANTUM|BACKEND_TORCH" docs/source/adv_install.rst
```
For each row write the concrete finding (e.g. `not on master; in open PR #967`, or `dissolved — block uses @loader_path, no ~`, or `present on master @ <sha>`). No cell may stay blank.

- [ ] **Step 5: Fill `macOS-verifiable?` and `Classification`**

- `macOS-verifiable?`: `yes (native)` for docs/macOS-BLAS/pip-doc items; `yes (cibuildwheel Docker)` for manylinux-only items like 0.3; `no (Linux+CUDA)` for #948/CUDA-runtime items; `n/a` for strategy row S.
- `Classification`: apply the definitions from the header. Expected anchors from brainstorming (verify, don't assume): 0.1→`covered` (PR #967); 1.1/1.3→`covered` (PR #950); 1.2→`dissolved`; 1.5/#948→`gap-decision-coupled`; 0.3→`gap`; 2.x/3.x packaging & S→`superseded` (reframed by #969) except concrete gaps.

- [ ] **Step 6: Add the "First-PR candidate" section**

Append a section that lists which items pass each of the three gates and names exactly one first-PR candidate:
```markdown
## First-PR candidate

Gates (all required): (1) classification `gap`; (2) macOS-verifiable; (3) not `gap-decision-coupled` on #969.

| Item | (1) gap? | (2) macOS-verifiable? | (3) not #969-coupled? | Passes all? |
|------|----------|-----------------------|-----------------------|-------------|
| #948 CUDA rpath | yes | no | no | ✗ |
| 0.3 OpenBLAS dedup | yes | <verdict from Step 5> | yes | <✓/✗> |
| ... | | | | |

**Selected first PR:** <the single item passing all three, with a one-line why, or —
if none passes — an explicit "no macOS-verifiable non-coupled gap; first PR requires
CI/Linux, recommend X" statement.>
```

- [ ] **Step 7: Run the acceptance checks**

Run:
```bash
# every inventory item present (expect 18 data rows: 0.1..3.5 + S)
grep -cE "^\| [0-9S]" docs/08-upstream-reconciliation.md
# no unfilled cells left in the main table (no empty ' | |  |' runs)
grep -nE "\|\s*\|\s*\|" docs/08-upstream-reconciliation.md || echo "no-empty-cells"
# first-PR section exists and names a selection
grep -q "Selected first PR:" docs/08-upstream-reconciliation.md && echo OK
```
Expected: count ≥ 18; `no-empty-cells`; `OK`.

- [ ] **Step 8: Commit**

```bash
git add docs/08-upstream-reconciliation.md
git commit -m "docs: add chapter 08 upstream reconciliation map"
```

---

### Task 3: Draft and file the umbrella tracking issue

**Files:**
- Create: `scratch/umbrella-issue-body.md` (uncommitted draft; the repo already ignores `scratch/`-style cruft — never `git add` it)

**Interfaces:**
- Consumes: the classifications and first-PR candidate from Task 2.
- Produces: a live issue on `Cytnx-dev/Cytnx`; its URL/number is recorded for Task 4.

- [ ] **Step 1: Draft the issue body to a scratch file**

Write the body to `scratch/umbrella-issue-body.md` (create the `scratch/` dir if needed; it stays uncommitted). It must be self-contained (no links into this private repo) and reference — not restate — the cluster. Structure:
```markdown
# Deployment strategy — status board

Umbrella tracker grouping the existing build/packaging issues into a phased
deployment roadmap. This references existing issues; it does not duplicate them.

## Phase gate — packaging model (Phases 2–3 depend on this)
- [ ] Decide PyTorch-style single-package vs JAX-style cpu/cuda12/cuda13 variants — #969 (+ #966)

## Phase 0 — wheel size / PyPI quota
- [ ] Exclude C++ SDK from the Python wheel — #947 (PR #967)
- [ ] Dedup OpenBLAS across manylinux wheels — #947  ← open gap

## Phase 1 — build-time discovery
- [ ] Harden cuTENSOR/cuQuantum finders + CUDA diagnostics — #945/#946/#949 (PR #950)
- [ ] rpath-pin installed CUDA extension to build toolkit — #948  ← open gap (decision-coupled)

## Phase 2 — packaging & channels
- [ ] pip/uv + conda-forge story — reframed by #969

## Phase 3 — GPU accelerators
- [ ] CUDA CI + conda-forge GPU deps — reframed by #969
```
(Fill exact issue/PR numbers and gap flags from Task 2's finalized map.)

- [ ] **Step 2: Acceptance check — no private-repo leakage in the draft**

Run:
```bash
grep -nE "cytnx_design_deployment|docs/0[0-9]-|pcchen/cytnx-deployment" scratch/umbrella-issue-body.md && echo "LEAK-FOUND" || echo "clean"
```
Expected: `clean` (the issue must not reference the private analysis repo).

- [ ] **Step 3: Get user approval before posting (outward-facing action)**

Show the drafted body to the user and get explicit approval to file it on `Cytnx-dev/Cytnx`. Do not run Step 4 until approved. If the user edits, re-run Step 2.

- [ ] **Step 4: File the issue**

Run:
```bash
gh issue create -R Cytnx-dev/Cytnx \
  --title "Deployment strategy — status board" \
  --body-file scratch/umbrella-issue-body.md
```
Expected: prints the new issue URL. Record the number as `<UMBRELLA_ISSUE>`.

- [ ] **Step 5: Verify the issue is live and well-formed**

Run:
```bash
gh issue view <UMBRELLA_ISSUE> -R Cytnx-dev/Cytnx --json title,body --jq '.title, (.body|length)'
```
Expected: title `Deployment strategy — status board` and a non-zero body length. (No repo commit in this task — the artifact is the upstream issue.)

---

### Task 4: Wire chapter 08 + the tracking issue into the repo index

**Files:**
- Modify: `README.md`
- Modify: `docs/08-upstream-reconciliation.md` (add the umbrella issue link at top)

**Interfaces:**
- Consumes: chapter 08 (Task 2), `<UMBRELLA_ISSUE>` URL (Task 3).
- Produces: README "How to read this" lists chapter 08; the Status section points at the live tracking issue.

- [ ] **Step 1: Write the acceptance check and run it (expect fail)**

Run:
```bash
grep -q "08-upstream-reconciliation" README.md && echo LINKED || echo MISSING
```
Expected: `MISSING`.

- [ ] **Step 2: Add chapter 08 to the README reading list**

In `README.md`, under "How to read this", add after the chapter 07 line:
```markdown
9. [Upstream reconciliation & status board](docs/08-upstream-reconciliation.md)
```

- [ ] **Step 3: Update the README Status section**

Replace the existing Status paragraph with one that reflects this sub-project:
```markdown
## Status

Analysis complete. Implementation campaign underway: the ch.06/07 roadmap is
reconciled against upstream in [chapter 08](docs/08-upstream-reconciliation.md),
and tracked upstream at Cytnx-dev/Cytnx#<UMBRELLA_ISSUE>. The first gap PR is
named in chapter 08's "First-PR candidate" section; each phase is a separate
spec → plan → PR cycle.
```

- [ ] **Step 4: Add the tracking-issue link to the top of chapter 08**

Under the chapter 08 header line, add:
```markdown
> Upstream tracking issue: Cytnx-dev/Cytnx#<UMBRELLA_ISSUE>
```

- [ ] **Step 5: Run the acceptance checks**

Run:
```bash
grep -q "08-upstream-reconciliation" README.md && echo LINKED
grep -q "Cytnx-dev/Cytnx#" README.md docs/08-upstream-reconciliation.md && echo REF-OK
```
Expected: `LINKED` and `REF-OK`.

- [ ] **Step 6: Commit**

```bash
git add README.md docs/08-upstream-reconciliation.md
git commit -m "docs: link chapter 08 and upstream tracking issue from README"
```
