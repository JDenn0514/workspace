<!-- filename: 2026-04-17-sgp-input-hardening.md -->

# 2026-04-17 — SGP Input Hardening: pool_baseline validation + warning class split

> Run: `sgp-input-hardening-2026-04-17` | Profile: r-package | Verdict: PASS

## What Changed

Hardened input validation in `sgp()` with two surgical changes: (1) a new top-of-function
`pool_baseline` check that aborts with `rotostats_error_invalid_pool_baseline` if anything
other than `"projection_pool"` is passed; (2) a warning class split that replaces the
former dual-use `rotostats_warning_missing_category_column` with two distinct classes —
the original class is retained for the "column absent" condition, and a new
`rotostats_warning_zero_playing_time` class is used for the "player has 0/NA IP or AB"
condition. No algorithmic change; SGP math is unchanged.

The documentation pass (this scriber run) regenerated `man/sgp.Rd` from the updated roxygen
source. This was deferred from the main feature branch merge (PR #5) and completed here.

## Files Changed

| File | Action | Description |
|------|--------|-------------|
| `R/sgp.R` | modified | Builder `a1a56e1`: new Step 1b validation block; three call-site class changes; updated `@param pool_baseline`, `@section Warnings:`, `@details Validation order` roxygen |
| `tests/testthat/test-sgp.R` | modified | Tester `fc6f980`: updated zero-IP and zero-AB test expectations; added 6+ new tests for pool_baseline abort and warning class isolation |
| `tests/testthat/test-sgp-integration.R` | modified | Tester `fc6f980`: updated TS-11 (line 572) and TS-12 (line 625) to expect `rotostats_warning_zero_playing_time`; TS-10 (line 520) retained `rotostats_warning_missing_category_column` |
| `plans/error-messages.md` | modified | Builder `a1a56e1`: new `rotostats_error_invalid_pool_baseline` error row; narrowed `rotostats_warning_missing_category_column` description; new `rotostats_warning_zero_playing_time` warning row |
| `ARCHITECTURE.md` | modified | Builder `a1a56e1`: updated J5 node label and added J5a node in call graph; replaced KDD item 3 dual-use prose with split-class description |
| `NEWS.md` | modified | Builder `a1a56e1`: added `## Breaking changes` section with two bullets describing the new abort and warning class migration |
| `man/sgp.Rd` | modified (regenerated) | Scriber (this run): rebuilt via `devtools::document()` to reflect updated roxygen |
| `plans/sgp-cleanup.md` | created | OOB commit `ce1fbde`: project planning doc for follow-up tasks (not part of this run's scope) |

---

## Process Record

This section captures the full workflow history for `sgp-input-hardening-2026-04-17`.

### Proposal (from planner)

**Implementation spec summary** (from `spec.md`):

- Two surgical changes to `R/sgp.R`; no algorithmic change, no new arguments.
- **Change 1 — `pool_baseline` validation**: Insert a new Step 1b validation block between
  line 219 (end of Step 1: column normalization) and line 222 (start of Step 2:
  `rate_conversion` validation). The block checks `pool_baseline %in% "projection_pool"` and
  aborts with `rotostats_error_invalid_pool_baseline`. Placement ensures the check fires
  before all other validation regardless of `rate_conversion`.
- **Change 2 — Warning class split**: Replace the `class =` argument at three `cli::cli_warn()`
  call sites (lines 539, 572, 603 — ERA zero-IP, WHIP zero-IP, AVG zero-AB) from
  `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time`. Line 342
  (column-absent case, Step 8) is explicitly **not changed**.
- Roxygen updates: `@param pool_baseline`, `@section Warnings:` (split into two `\subsection{}`
  blocks), `@details Validation order` (5-item list expanded to 6 items with `pool_baseline` as
  item 1).
- `plans/error-messages.md` receives three surgical edits: one new error row, one narrowed
  warning row, one new warning row.
- `ARCHITECTURE.md` receives two surgical edits: J5 node label updated, J5a node added,
  KDD item 3 dual-use prose replaced.
- `NEWS.md` receives a `## Breaking changes` section with two bullets.
- `man/sgp.Rd` regeneration deferred to scriber via `devtools::document()`.

**Test spec summary** (from `test-spec.md` — read by tester, summarised here):

- **Behavioral contracts**: 6 contracts to verify: (1) default path unchanged; (2) any
  non-`projection_pool` `pool_baseline` aborts with correct class before other checks;
  (3) missing column fires only `rotostats_warning_missing_category_column`; (4) zero playing
  time fires only `rotostats_warning_zero_playing_time`; (5) no class bleed between conditions;
  (6) all existing tests pass.
- **New tests** (§3): pool_baseline abort for "foo", "per_player", "universal_constants";
  mutual exclusivity of the two warning classes; validation order (pool_baseline before
  rate_conversion).
- **Updated tests** (§4–§5): test-sgp.R lines 488 and 517; integration test lines 572 and 625.
- **Tolerances**: all class assertions are exact (no numerical tolerance); existing numerical
  tests use `1e-12` or `1e-6` as before.

### Implementation Notes (from builder)

- **Step numbering "1b"**: Preserved existing step numbers (2–15) by naming the new block
  "Step 1b". Makes it clear this runs after column normalization but before any method dispatch.
- **`\code{}` syntax in roxygen validation order**: New items in the `@details Validation order`
  list use `\code{}` instead of backticks for consistency with the spec's replacement text.
- **J5a node in diagram**: Spec called for both updating J5 (legacy "zero IP/AB" context) and
  adding J5a (clean class-name-only node). Both retained per spec §4 Edit 1.
- **Downstream stubs unreachable**: After Step 1b is inserted, the downstream
  `rotostats_error_missing_config_field` path for `pool_baseline = "per_player"` or
  `"universal_constants"` becomes unreachable dead code. Not removed here — guarded by the
  new gate. Future expansion: widen `valid_pool_baselines` vector in Step 1b only.
- **No test files modified by builder**: Per spec §7, test files are tester's write surface.
  Builder committed `a1a56e1` touching only `R/sgp.R`, `plans/error-messages.md`,
  `ARCHITECTURE.md`, and `NEWS.md`.

### Validation Results (from tester)

**Per-Test Result Table** (from `audit.md`):

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|------|--------|----------|--------|-----------|------------|---------|
| §3a pool_baseline="foo" | error class | `rotostats_error_invalid_pool_baseline` | `rotostats_error_invalid_pool_baseline` | exact | — | PASS |
| §3a pool_baseline="per_player" | error class | `rotostats_error_invalid_pool_baseline` | `rotostats_error_invalid_pool_baseline` | exact | — | PASS |
| §3a pool_baseline="universal_constants" | error class | `rotostats_error_invalid_pool_baseline` | `rotostats_error_invalid_pool_baseline` | exact | — | PASS |
| §3b missing column | warning class | `rotostats_warning_missing_category_column` | `rotostats_warning_missing_category_column` | exact | — | PASS |
| §3b missing column | NOT zero_playing_time fires | zero_playing_time should NOT fire | zero_playing_time did NOT fire | exact | — | PASS |
| §3c zero IP | warning class | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | exact | — | PASS |
| §3c zero IP | NOT missing_category_column fires | missing_category_column should NOT fire | missing_category_column did NOT fire | exact | — | PASS |
| §4a zero-IP ERA (test-sgp.R:488) | warning class | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | exact | — | PASS |
| §4a zero-IP ERA (test-sgp.R:502) | sgp_ERA[2] | NA | NA | exact | — | PASS |
| §4b zero-AB AVG (test-sgp.R:517) | warning class | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | exact | — | PASS |
| §4b zero-AB AVG (test-sgp.R:529) | sgp_AVG[2] | NA | NA | exact | — | PASS |
| §5a TS-10 (line 520) | warning class | `rotostats_warning_missing_category_column` | `rotostats_warning_missing_category_column` | exact | — | PASS |
| §5a TS-10 | sgp_SB all NA | TRUE | TRUE | exact | — | PASS |
| §5b TS-11 (line 572) | warning class | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | exact | — | PASS |
| §5b TS-11 | sgp_ERA[4] | NA | NA | exact | — | PASS |
| §5b TS-11 | sgp_WHIP[4] | NA | NA | exact | — | PASS |
| §5c TS-12 (line 625) | warning class | `rotostats_warning_zero_playing_time` | `rotostats_warning_zero_playing_time` | exact | — | PASS |
| §5c TS-12 | sgp_AVG[4] | NA | NA | exact | — | PASS |
| §6 invariant — validation order | error class when pool_baseline invalid + rate_conversion invalid | `rotostats_error_invalid_pool_baseline` | `rotostats_error_invalid_pool_baseline` | exact | — | PASS |

Summary: 19 tests executed, 19 passed, 0 failed.

**Before/After Comparison Table** (from `audit.md`):

| Metric | Before (develop HEAD) | After (feature branch) | Change | Interpretation |
|--------|-----------------------|------------------------|--------|----------------|
| Warning class for zero-IP ERA/WHIP rows | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` | Class renamed | Correct — zero playing time is not a missing column |
| Warning class for zero-AB AVG rows | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` | Class renamed | Correct — zero playing time is not a missing column |
| Warning class for column-absent rows | `rotostats_warning_missing_category_column` | `rotostats_warning_missing_category_column` | No change | Correct — column-absent case retains original class |
| `pool_baseline = "foo"` | No error (parameter not validated) | Aborts with `rotostats_error_invalid_pool_baseline` | New validation | Correct — invalid input now caught early |
| `pool_baseline = "projection_pool"` (default) | Computed correctly | Computed correctly | No change | Regression-free |
| NA propagation for zero-IP/AB players | NA returned | NA returned | No change | Regression-free |
| devtools::check() NOTE count | 5 | 5 | 0 | No new NOTEs introduced |

Additional notes:

- `devtools::test(filter = "sgp")`: `[ FAIL 0 | WARN 125 | SKIP 0 | PASS 246 ]`. The 125 WARNs
  are all within pre-existing sgp_denominators calibration tests and are expected behavior.
- `devtools::check()` on feature branch: `0 errors | 0 warnings | 5 notes`.
- `devtools::check()` on develop HEAD (baseline): `0 errors | 0 warnings | 5 notes`.
- NOTEs are identical on both branches — no regressions.
- Scriber re-ran `devtools::check(document = FALSE, args = c('--no-manual'))` on
  `feature/sgp-input-hardening-docs` after `devtools::document()`: same result,
  `0 errors | 0 warnings | 5 notes`.

### Problems Encountered and Resolutions

| # | Problem | Signal | Routed To | Resolution |
|---|---------|--------|-----------|------------|
| — | — | — | — | — |

No problems encountered. Zero HOLD rounds in planner. No BLOCK or STOP signals in any
teammate phase. Builder's implementation was complete and fully spec-compliant on first pass.
Tester's audit returned PASS on first run.

**Out-of-band event (documented for audit trail, not a problem):** The feature + test commits
for this run (`a1a56e1` and `fc6f980`) were merged to `develop` ahead of schedule via PR #5
(`docs/cleanup-plans`) before scriber could run. `man/sgp.Rd` was therefore NOT regenerated
during that merge. The `feature/sgp-input-hardening-docs` branch was cut from `develop` after
the PR merged (at `3d2bd69`) specifically to allow scriber to complete the documentation pass.
This is an unusual but fully recovered situation — no data loss, no specification deviation.

### Review Summary

Pending — reviewer review follows scriber.

- **Pipeline isolation**: pending
- **Convergence**: N/A — this is an input-validation change, not a statistical algorithm
- **Tolerance integrity**: pending
- **Verdict**: pending

---

## Design Decisions

1. **`pool_baseline` validation placed before `rate_conversion` validation (Step 1b)**: The
   `pool_baseline` check is inserted after column-name normalization (Step 1) and before all
   other validation. This ensures it fires regardless of `rate_conversion` value — even if
   `rate_conversion` is invalid, an invalid `pool_baseline` produces the most specific error
   message first. The spec explicitly required "catches invalid `pool_baseline` before
   `rate_conversion` check" (verified by tester's §6 invariant test). The placement also gates
   the documented-but-unimplemented values (`"per_player"`, `"universal_constants"`) at a single
   authoritative point, rendering their downstream stubs unreachable.

2. **Warning class split rationale**: The former dual-use `rotostats_warning_missing_category_column`
   covered two logically distinct failure modes: (a) a column entirely absent from `projections`
   (a data-completeness problem) and (b) a player with 0/NA IP or AB on a scored rate stat (a
   data-quality problem). These warrant separate `withCallingHandlers` targets because a caller
   who wants to suppress zero-playing-time warnings for pure hitters (expected condition) should
   not also suppress missing-column warnings (unexpected condition). The split gives callers
   fine-grained control with no change to the warning messages themselves (only the `class =`
   argument at three call sites changed).

3. **Step numbering "1b" instead of renumbering**: Preserving all existing step numbers (Step 2
   through Step 15) avoids comment drift in a 637-line function that is already well-annotated.
   The "1b" label makes the insertion point and purpose immediately clear to anyone reading the
   code chronologically. The spec explicitly required "1b" naming.

---

## Handoff Notes

**For the next developer working on `sgp()`:**

1. **`withCallingHandlers` migration required for zero-playing-time callers**: Any downstream
   code that uses `withCallingHandlers(rotostats_warning_missing_category_column = ...)` to
   intercept zero-playing-time rows (0/NA IP for ERA/WHIP or 0/NA AB for AVG) must be updated
   to `withCallingHandlers(rotostats_warning_zero_playing_time = ...)`. The `NEWS.md` breaking
   changes section documents this with the exact call pattern. This is a **minor breaking
   change** for callers using `withCallingHandlers` — users catching only the error hierarchy
   (via `tryCatch`) are unaffected.

2. **Downstream stubs are now dead code**: The Step 6b code path that previously reached
   `rotostats_error_missing_config_field` for `pool_baseline = "per_player"` or
   `"universal_constants"` is now unreachable (Step 1b gates those values first). When
   implementing support for additional `pool_baseline` values, expand the `valid_pool_baselines`
   vector in Step 1b — that is the single authoritative gate. The downstream stubs can then be
   retained or removed at that time.

3. **`man/sgp.Rd` is now fully regenerated and consistent with `R/sgp.R`**: The Rd file was
   stale after the out-of-band PR #5 merge. This scriber run regenerated it via
   `devtools::document()`. Future roxygen edits to `R/sgp.R` must be followed by
   `devtools::document()` before committing. Per the r-package profile, scriber owns this step.

4. **The 5 pre-existing NOTEs are not regressions**: (a) `.playwright-mcp` hidden directory,
   (b) `unable to verify current time`, (c) non-standard top-level files, (d) NEWS.md no
   entries found, (e) Escaped LaTeX specials in replacement Rd files. These are all
   infrastructure-level issues unrelated to `sgp()`.

5. **Follow-up tasks**: See `plans/sgp-cleanup.md` for the R2/R3 follow-on tasks created in
   the out-of-band `ce1fbde` commit.
