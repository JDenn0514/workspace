# Review — sgp-input-hardening-2026-04-17

**Reviewer:** Claude Sonnet 4.6
**Date:** 2026-04-17
**Commits reviewed:** a1a56e1 (builder), fc6f980 (tester), 240f683 (scriber)
**Branch at review:** develop (all commits landed on origin/develop)

---

## Verdict

**PASS WITH NOTE**

The merged state on origin/develop satisfies all four acceptance criteria from request.md.
Both pipelines independently converged on the same behavioral contract. Two workflow
protocol violations occurred prior to this review; the user has explicitly accepted both.
The run may be marked DONE after workspace-sync.

---

## Workflow Violations (Documented, Not Blocking)

### Violation 1 — Feature branch merged to develop ahead of schedule

Builder commit `a1a56e1` and tester commit `fc6f980` merged to `develop` via PR #5
(`docs/cleanup-plans`) while this run was mid-flight, before scriber could run
`devtools::document()`. As a result, `man/sgp.Rd` was stale on develop for a window
between the PR merge and scriber's recovery commit. No data loss occurred; the recovery
was clean. User accepted this state.

**Impact on verdict:** None. The final state on develop is correct.

### Violation 2 — Scriber pushed directly to origin/develop

Scriber commit `240f683` (regenerated `man/sgp.Rd` + cosmetic ARCHITECTURE.md tweaks) was
pushed directly to `origin/develop` rather than via a feature branch and PR. The workflow
protocol requires all commits go through a feature branch. User was asked and chose "Accept
state, run reviewer + sync."

**Impact on verdict:** None. The commit content is correct and innocuous (doc regeneration
only). Flagged for process record.

---

## Step 1 — Comprehension Foundation

- `comprehension.md` exists with verdict **FULLY UNDERSTOOD**. Zero HOLD rounds.
- All call sites identified correctly (lines 342, 539, 572, 603 of pre-change file; 373,
  570, 603, 634 post-insertion).
- Formulas in `comprehension.md` match `spec.md` exactly.
- The one judgment call (downstream stub reachability) is explicitly documented and resolved
  per the dispatch prompt.

**Status: CLEARED**

---

## Step 2 — Pipeline Isolation

Builder's `implementation.md` does not reference `test-spec.md` or `audit.md`.
Builder explicitly noted "builder does not touch test files (that is tester's write surface)"
with spec.md §7 as the authority — derived from `spec.md`, not from `test-spec.md`.

Tester's `audit.md` references `test-spec.md` (its own input spec) and does not reference
`spec.md` or `implementation.md`. Tester independently verified the behavioral contracts
from `test-spec.md`.

Builder's write surface: `R/sgp.R`, `plans/error-messages.md`, `ARCHITECTURE.md`, `NEWS.md`.
Tester's write surface: `tests/testthat/test-sgp.R`, `tests/testthat/test-sgp-integration.R`.
No overlap.

**Status: CLEARED — isolation maintained**

---

## Step 3 — Cross-Specification Comparison

| Area | spec.md | test-spec.md | Alignment |
|------|---------|--------------|-----------|
| New error class name | `rotostats_error_invalid_pool_baseline` | `rotostats_error_invalid_pool_baseline` | MATCH |
| pool_baseline valid value | `"projection_pool"` only | `"projection_pool"` only | MATCH |
| pool_baseline fires before rate_conversion | Yes (spec §2a rationale) | Yes (test-spec §6 invariant) | MATCH |
| New warning class name | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | MATCH |
| Line 342 — unchanged | No change (spec §2b table) | No change (test-spec §5a) | MATCH |
| Lines 539/572/603 — class changed | Yes (spec §2b table) | Yes (test-spec §4a/4b/§5b/§5c) | MATCH |
| "AND NOT" assertions required | Not mentioned (builder scope only) | Explicitly required (test-spec §3b, §3c) | Consistent |
| Tolerance for class comparisons | Exact string match (spec §8) | Exact string match (test-spec §2) | MATCH |

No gaps found between spec.md and test-spec.md.

**Status: CLEARED — specifications consistent**

---

## Step 4 — Convergence Verification

### Per-Test Result Table

audit.md contains a complete Per-Test Result Table with 19 rows covering all scenarios
from test-spec.md §3, §4, and §5 plus the §6 invariant. All 19 rows show PASS.

Cross-referencing test-spec.md scenarios against the per-test table:

| test-spec.md Scenario | In Per-Test Table | Verdict |
|-----------------------|-------------------|---------|
| §3a pool_baseline="foo" | Yes (row 1) | PASS |
| §3a pool_baseline="per_player" | Yes (row 2) | PASS |
| §3a pool_baseline="universal_constants" | Yes (row 3) | PASS |
| §3b missing column fires missing-column | Yes (rows 4-5) | PASS |
| §3c zero IP fires zero-playing-time | Yes (rows 6-7) | PASS |
| §4a zero-IP ERA class update | Yes (row 8) | PASS |
| §4a zero-IP ERA NA propagation | Yes (row 9) | PASS |
| §4b zero-AB AVG class update | Yes (row 10) | PASS |
| §4b zero-AB AVG NA propagation | Yes (row 11) | PASS |
| §5a TS-10 (no change) | Yes (rows 12-13) | PASS |
| §5b TS-11 class update | Yes (rows 14-16) | PASS |
| §5c TS-12 class update | Yes (rows 17-18) | PASS |
| §6 validation order invariant | Yes (row 19) | PASS |

### Before/After Comparison Table

audit.md contains a Before/After Comparison Table covering all observable changes.
No algorithmic changes occurred; all rows are class-rename or new-validation changes.
No metrics worsened without justification.

**Status: CLEARED — complete per-test table, complete before/after table, all PASS**

---

## Step 5 — Test Coverage

### Changed code paths vs. test coverage

| Code change | Covered by test | Assertion type |
|-------------|-----------------|----------------|
| Step 1b pool_baseline validation block (new, ~8 lines) | Yes — §3a tests for "foo", "per_player", "universal_constants"; §6 invariant for ordering | Correctness (error class exact match) |
| Line 373 (formerly 342) Step 8 — class unchanged | Yes — §3b AND NOT assertion; §5a TS-10 | Correctness (class exact match + no-fire assertion) |
| Line 570 (formerly 539) Step 14a ERA | Yes — §3c AND NOT assertion; §4a; §5b TS-11 | Correctness (class exact match + NA propagation) |
| Line 603 (formerly 572) Step 14c WHIP | Yes — §5b TS-11 | Correctness (class exact match + NA propagation) |
| Line 634 (formerly 603) Step 14d AVG | Yes — §3c; §4b; §5c TS-12 | Correctness (class exact match + NA propagation) |
| Roxygen @param pool_baseline | Verified in man/sgp.Rd (docs check) | Documentation fidelity |

All changed code paths have independent test coverage.

### Assertion depth

Tester's assertions are correctness-level (exact class string match, NA propagation
verified, "AND NOT" firing verified via withCallingHandlers inspection). No structural-only
assertions.

**Status: CLEARED — all changed paths covered with correctness assertions**

---

## Step 6 — Structural Refactors

Not applicable. This change is a pure insertion (11 lines) plus three `class =` argument
flips. No structural refactoring occurred. No closures promoted, no state capture changes.

**Status: N/A**

---

## Step 7 — Validation Evidence

### Commands actually run

audit.md records two exact command invocations with output:

1. `devtools::test(filter = "sgp")` — result: `[ FAIL 0 | WARN 125 | SKIP 0 | PASS 246 ]`
2. `devtools::check(document = FALSE, args = c('--no-manual'))` — on feature branch: 0/0/5
3. `devtools::check(...)` — on develop HEAD baseline: 0/0/5

The 125 WARNs in the test run are pre-existing sgp_denominators calibration warnings;
audit.md explicitly identifies them as expected. This is consistent with the test suite
behavior for this package.

### All test-spec.md scenarios executed

Verified above in Step 4. 19/19 scenarios executed, all PASS.

### ERRORs and WARNINGs

0 ERRORs, 0 WARNINGs in R CMD check. All 5 NOTEs are pre-existing infrastructure notes
(also present on develop HEAD baseline). NOTE delta = 0.

### Step 7a — Tolerance Integrity Audit

All class comparisons in audit.md use exact string match — consistent with test-spec.md §2
policy of "exact string match." No numerical tolerances were modified. The NA propagation
assertions use `expect_true(is.na(...))` — no tolerance involved.

audit.md explicitly states: "No tolerance values were modified."

No tolerance inflation detected. No evasion patterns detected (no try/catch wrappers
suppressing failures, no reduced sample sizes, no silent omissions).

**Status: CLEARED — exact tolerances throughout, no inflation**

---

## Step 8 — Documentation and Process Record

### ARCHITECTURE.md

- Present in target repo root: `/Users/jacobdennen/rotostats/ARCHITECTURE.md` — CONFIRMED
- Present in run directory: `sgp-input-hardening-2026-04-17/ARCHITECTURE.md` — CONFIRMED
- Both contain Mermaid diagrams — CONFIRMED
- J5a node added (`cli_warn\nrotostats_warning_zero_playing_time`) — CONFIRMED (line 123)
- J5 node updated (`cli_warn\nrotostats_warning_zero_playing_time\nzero IP/AB`) — CONFIRMED (line 139)
- KDD item 3 replaced with split-class prose — CONFIRMED (ARCHITECTURE.md line 446)
- Highlight styles updated for this run's changed nodes — CONFIRMED

### log-entry.md

- Present in run directory — CONFIRMED
- Contains `<!-- filename: 2026-04-17-sgp-input-hardening.md -->` header — CONFIRMED
- Contains: What Changed, Files Changed, Process Record (with Per-Test Result Table,
  Before/After Comparison Table, Problems and Resolutions), Design Decisions, Handoff Notes — CONFIRMED
- Out-of-band event documented in Problems/Resolutions — CONFIRMED

### Target repo cleanliness

`ARCHITECTURE.md` — present, expected.
`HANDOFF.md` — present in target repo root. This is a PRE-EXISTING artifact from
2026-04-10, predating this run. It was present on develop before this run started
and is listed in the pre-existing R CMD check NOTE for non-standard top-level files.
It was NOT introduced by this run.
`CHANGELOG.md` — absent. CONFIRMED.
`runs/` directory — absent. CONFIRMED.
`log/` directory — absent. CONFIRMED.

The `HANDOFF.md` pre-existence is already tracked in the R CMD check NOTEs as a
known infrastructure issue unrelated to this change.

### man/sgp.Rd

Regenerated by scriber commit 240f683 via `devtools::document()`. Verified:
- `pool_baseline` argument documents abort with `rotostats_error_invalid_pool_baseline` — CONFIRMED (lines 46-51)
- Validation order section shows 6-item list with pool_baseline as item 1 — CONFIRMED (lines 137-142)
- Warnings section shows two `\subsection{}` blocks, one per class — CONFIRMED (lines 178-194)

### news.md

`## Breaking changes` section present with two bullets exactly matching spec.md §5 — CONFIRMED

### plans/error-messages.md

- New error row `rotostats_error_invalid_pool_baseline` — CONFIRMED (line 32)
- Narrowed `rotostats_warning_missing_category_column` row — CONFIRMED (line 69)
- New `rotostats_warning_zero_playing_time` row — CONFIRMED (line 70)
- No other rows removed or reordered — CONFIRMED (table structure intact)

**Status: CLEARED — all documentation correct and complete**

---

## Acceptance Criteria Check (from request.md)

| Criterion | Verified | Evidence |
|-----------|----------|----------|
| `sgp(..., pool_baseline = "projection_pool")` behaves as before | PASS | All 246 existing tests pass; Before/After table shows no change on default path |
| Any other pool_baseline aborts with `rotostats_error_invalid_pool_baseline` naming valid values | PASS | §3a tests for "foo"/"per_player"/"universal_constants" all PASS; Step 1b code confirmed in R/sgp.R:239-250 |
| Missing category fires `rotostats_warning_missing_category_column` AND NOT zero-playing-time | PASS | §3b AND NOT test; TS-10 line 520 unchanged |
| IP=0 fires `rotostats_warning_zero_playing_time` AND NOT missing-column | PASS | §3c AND NOT test; TS-11 line 572 and TS-12 line 625 updated |
| R CMD check: 0E/0W/NOTEs unchanged | PASS | 0E/0W/5N on feature branch; 0E/0W/5N on develop baseline; delta = 0 |
| All existing sgp tests still pass | PASS | FAIL 0, PASS 246, SKIP 0 |

All four acceptance criteria from request.md are satisfied.

---

## Artifact Cross-Check Table

| Artifact | Expected | Actual | Match |
|----------|----------|--------|-------|
| R/sgp.R Step 1b present | Lines 239-250 with `rotostats_error_invalid_pool_baseline` | CONFIRMED | YES |
| R/sgp.R line 373 class | `rotostats_warning_missing_category_column` (unchanged) | CONFIRMED | YES |
| R/sgp.R line 570 class | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| R/sgp.R line 603 class | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| R/sgp.R line 634 class | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| plans/error-messages.md new error row | `rotostats_error_invalid_pool_baseline` | CONFIRMED (line 32) | YES |
| plans/error-messages.md narrowed warning row | Column-absent only | CONFIRMED (line 69) | YES |
| plans/error-messages.md new warning row | `rotostats_warning_zero_playing_time` | CONFIRMED (line 70) | YES |
| NEWS.md Breaking changes section | Two bullets per spec.md §5 | CONFIRMED (lines 50-68) | YES |
| ARCHITECTURE.md J5 node updated | `rotostats_warning_zero_playing_time\nzero IP/AB` | CONFIRMED (line 139) | YES |
| ARCHITECTURE.md J5a node added | `rotostats_warning_zero_playing_time` | CONFIRMED (line 123) | YES |
| ARCHITECTURE.md KDD item 3 replaced | Split-class description | CONFIRMED (line 446) | YES |
| man/sgp.Rd pool_baseline param | Documents abort with new class | CONFIRMED (lines 46-51) | YES |
| man/sgp.Rd validation order | 6-item list, pool_baseline first | CONFIRMED (lines 137-142) | YES |
| man/sgp.Rd Warnings section | Two subsection blocks | CONFIRMED (lines 178-194) | YES |
| test-sgp.R line 488 class | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| test-sgp.R line 517 class | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| test-sgp.R Section 12 new tests | pool_baseline abort (3 values) + AND NOT assertions | CONFIRMED | YES |
| test-sgp.R Section 13 new test | Validation order invariant | CONFIRMED | YES |
| test-sgp-integration.R line 520 | `rotostats_warning_missing_category_column` (unchanged) | CONFIRMED | YES |
| test-sgp-integration.R line 572 | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |
| test-sgp-integration.R line 625 | `rotostats_warning_zero_playing_time` | CONFIRMED | YES |

---

## Self-Check Before PASS

- [x] Verified comprehension.md exists with FULLY UNDERSTOOD verdict (step 1)
- [x] Verified pipeline isolation (step 2)
- [x] Cross-compared spec.md against test-spec.md (step 3)
- [x] Verified convergence between both pipelines (step 4)
- [x] Checked test coverage for every changed code path (step 5)
- [x] Assessed whether assertions are structural-only or correctness-level (step 5)
- [x] Step 6 N/A (no structural refactoring)
- [x] Verified tester ran required validation commands with exact evidence (step 7)
- [x] Verified tester executed ALL test-spec.md scenarios (step 7)
- [x] Cross-referenced ALL numerical tolerances in audit.md against test-spec.md — no inflation (step 7a)
- [x] Verified Per-Test Result Table present in audit.md with all scenarios covered (step 4)
- [x] Verified Before/After Comparison Table present in audit.md (step 4)
- [x] Step 5b N/A (not a simulation workflow)
- [x] Checked documentation, ARCHITECTURE.md in target repo root + run dir, log-entry.md in run dir (step 8)
- [x] Step 8b N/A (brain mode isolated, no brain-contributions.md)

---

## Notes for the Record

1. The pre-existing `HANDOFF.md` in the target repo root is not a regression from this run.
   It predates this run (2026-04-10) and is tracked in the existing R CMD check NOTE for
   non-standard top-level files. A cleanup task exists for it in `plans/sgp-cleanup.md`.

2. The NEWS.md note "No news entries found" R CMD check NOTE is pre-existing and is expected
   to resolve once the `rotostats (development version)` section format is recognized by the
   NEWS parser. Not introduced by this run.

3. This run's 246 passing tests (FAIL 0) includes all updated class assertions and all new
   tests. The 125 WARNs in the test run are pre-existing sgp_denominators calibration
   warnings, not new.

---

## Recommendation for Next Steps

The code is correct and all acceptance criteria are satisfied. Both workflow violations have
been accepted by the user and documented.

**Shipper action:** Run workspace-sync only. There is nothing to push to origin/develop
(240f683 is already there). The shipper should sync the run artifacts to the workspace repo
and mark the run DONE.

