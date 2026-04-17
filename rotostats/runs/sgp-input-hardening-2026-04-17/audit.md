# Audit — sgp-input-hardening-2026-04-17

**Tester:** tester agent (claude-sonnet-4-6)  
**Date:** 2026-04-17  
**Branch:** feature/sgp-input-hardening  
**Builder commit:** a1a56e1  
**Tester commit:** fc6f980  
**Verdict: PASS**

---

## Verdict Summary

| Check | Result |
|-------|--------|
| All new test scenarios from test-spec.md §3 | PASS |
| Existing test updates per test-spec.md §4–§5 | PASS |
| devtools::test(filter = "sgp") | PASS — 0 FAIL, 246 PASS, 0 SKIP |
| devtools::check() on feature/sgp-input-hardening | 0 errors, 0 warnings, 5 NOTEs |
| devtools::check() on develop HEAD (baseline) | 0 errors, 0 warnings, 5 NOTEs |
| NOTEs unchanged vs develop | PASS — identical 5 NOTEs, no new NOTEs |
| Regression: all existing tests pass | PASS |

---

## Environment

- R version 4.5.2 (2025-10-31) "Not Part in a Rumble"
- Platform: aarch64-apple-darwin20 (macOS Sequoia 15.4.1)
- Package: rotostats 0.0.0.9000

---

## Test-File Edits Made

Per test-spec.md §4 and §5, tester updated the following locations:

### test-sgp.R (§4a and §4b updates)

| Location | Change |
|----------|--------|
| Line 488 — zero-IP ERA test | Description updated; class changed from `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time` |
| Line 517 — zero-AB AVG test | Description updated; class changed from `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time` |

New sections added to test-sgp.R (per test-spec.md §3):

- **Section 12:** `pool_baseline = "foo"` aborts with `rotostats_error_invalid_pool_baseline` (3 tests: foo, per_player, universal_constants)
- **Section 12:** Missing category fires `missing_category_column` AND NOT `zero_playing_time`
- **Section 12:** Zero IP fires `zero_playing_time` AND NOT `missing_category_column`
- **Section 13:** Validation order — invalid `pool_baseline` fires before `rate_conversion` check

### test-sgp-integration.R (§5 updates)

| Location | Condition | Change |
|----------|-----------|--------|
| Line 520 (TS-10) | Column absent — condition (a) | NO CHANGE — `rotostats_warning_missing_category_column` retained (correct) |
| Line 572 (TS-11) | IP = 0 — condition (b) | Class changed from `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time` |
| Line 625 (TS-12) | AB = 0 — condition (b) | Class changed from `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time` |

---

## Per-Test Result Table

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

Note: Tolerances for numerical comparisons in existing tests use `1e-12` (exact arithmetic) or `1e-6` (floating point), consistent with test-spec.md §2. No tolerance values were modified.

---

## Before/After Comparison Table

This change is a class split and new validation — no algorithmic change. Key observable changes:

| Metric | Before (develop HEAD) | After (feature branch) | Change | Interpretation |
|--------|-----------------------|------------------------|--------|----------------|
| Warning class for zero-IP ERA/WHIP rows | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` | Class renamed | Correct — zero playing time is not a missing column |
| Warning class for zero-AB AVG rows | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` | Class renamed | Correct — zero playing time is not a missing column |
| Warning class for column-absent rows | `rotostats_warning_missing_category_column` | `rotostats_warning_missing_category_column` | No change | Correct — column-absent case retains original class |
| `pool_baseline = "foo"` | No error (parameter not validated) | Aborts with `rotostats_error_invalid_pool_baseline` | New validation | Correct — invalid input now caught early |
| `pool_baseline = "projection_pool"` (default) | Computed correctly | Computed correctly | No change | Regression-free |
| NA propagation for zero-IP/AB players | NA returned | NA returned | No change | Regression-free |
| devtools::check() NOTE count | 5 | 5 | 0 | No new NOTEs introduced |

---

## devtools::test() Output

**Command:** `/usr/local/bin/R --no-save -e "devtools::load_all('.'); devtools::test(filter = 'sgp')"`

**Result:**
```
[ FAIL 0 | WARN 125 | SKIP 0 | PASS 246 ]
```

The 125 WARNs are all emitted within existing sgp_denominators calibration tests and are expected/pre-existing behavior — they are not new and do not represent failures.

---

## devtools::check() on feature/sgp-input-hardening

**Command:** `/usr/local/bin/R --no-save -e "devtools::check(document = FALSE, args = c('--no-manual'), quiet = FALSE)"`

**Result:** `0 errors | 0 warnings | 5 notes`

**NOTEs:**
```
checking for hidden files and directories ... NOTE
  Found the following hidden files and directories:
    .playwright-mcp

checking for future file timestamps ... NOTE
  unable to verify current time

checking top-level files ... NOTE
  Non-standard files/directories found at top level:
    'ARCHITECTURE.md' 'HANDOFF.md' 'api_files' 'specs'

checking package subdirectories ... NOTE
  Problems with news in 'NEWS.md':
  No news entries found.

checking Rd files ... NOTE
  checkRd: (-1) replacement_from_prices.Rd:5: Escaped LaTeX specials: \$
  checkRd: (-1) replacement_from_prices.Rd:49: Escaped LaTeX specials: \$
  checkRd: (-1) replacement_from_prices.Rd:35: Escaped LaTeX specials: \$
  checkRd: (-1) replacement_level.Rd:157: Escaped LaTeX specials: \$
  checkRd: (-1) replacement_level.Rd:105: Escaped LaTeX specials: \$
  checkRd: (-1) replacement_level.Rd:109: Escaped LaTeX specials: \$
```

**Tests in check:** `Running 'testthat.R' [11s/11s] OK`

---

## devtools::check() on develop HEAD (baseline)

**Command:** `/usr/local/bin/R --no-save -e "devtools::check(document = FALSE, args = c('--no-manual'), quiet = FALSE)"` (run after `git checkout develop`)

**Result:** `0 errors | 0 warnings | 5 notes`

**NOTEs (identical to feature branch):**
- `.playwright-mcp` hidden file
- `unable to verify current time`
- Non-standard top-level files (`ARCHITECTURE.md`, `HANDOFF.md`, `api_files`, `specs`)
- `NEWS.md`: No news entries found
- Rd files: Escaped LaTeX specials in `replacement_from_prices.Rd` and `replacement_level.Rd`

---

## NOTE Comparison

| Branch | Errors | Warnings | NOTEs |
|--------|--------|----------|-------|
| develop HEAD | 0 | 0 | 5 |
| feature/sgp-input-hardening | 0 | 0 | 5 |
| **Delta** | **0** | **0** | **0** |

The feature branch introduces **no new NOTEs**. The 5 NOTEs on both branches are identical pre-existing infrastructure notes unrelated to this change.

Note on `NEWS.md` NOTE: This is a pre-existing NOTE because `NEWS.md` exists but has no parseable entries. The NEWS.md entry for this release (added by scriber) should resolve this once properly formatted. This NOTE is present on both branches and is not a regression.

---

## Regression Check

All pre-existing tests continued to pass:

- Section 1 (return structure): PASS
- Section 2 (counting-stat correctness): PASS
- Section 3 (rate-stat sign convention): PASS
- Section 4 (total_sgp additivity and NA propagation): PASS
- Section 5 (input validation order): PASS
- Section 6 (warning conditions — updated): PASS
- Section 7 (column name normalization): PASS
- Section 8 (blended-pool formula exactness): PASS
- Section 9 (league_history validation): PASS
- Section 10 (vectorization): PASS
- Section 11 (baseline year inform message): PASS
- TS-1 through TS-16 (integration tests): PASS
- EC-1 through EC-11 (edge case tests): PASS
- INV-1 through INV-7 (invariant tests): PASS

No regressions observed.

---

## Behavioral Contract Verification (test-spec.md §1)

| Contract | Verified |
|----------|----------|
| 1. `pool_baseline = "projection_pool"` produces same output as before | PASS — all existing tests pass |
| 2. Any other `pool_baseline` aborts with `rotostats_error_invalid_pool_baseline` before other checks | PASS — verified for "foo", "per_player", "universal_constants"; fires before rate_conversion check |
| 3. Missing category column fires `rotostats_warning_missing_category_column` ONLY | PASS — zero_playing_time does NOT fire for column-absent case |
| 4. Zero playing time fires `rotostats_warning_zero_playing_time` ONLY | PASS — missing_category_column does NOT fire for zero-IP/AB case |
| 5. No class bleed between the two warning conditions | PASS |
| 6. All existing tests pass | PASS — 0 FAIL |

