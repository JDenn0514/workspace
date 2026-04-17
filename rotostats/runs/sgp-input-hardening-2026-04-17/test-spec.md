# Test Spec — sgp-input-hardening-2026-04-17

**For:** Tester  
**Date:** 2026-04-17  
**Workflow:** Code-only (Workflow 2)  
**Branch:** `feature/sgp-input-hardening`

---

## 1. Behavioral Contract

After this change, `sgp()` must satisfy these observable behaviors:

1. **Valid `pool_baseline` — no change in behavior.** `sgp(..., pool_baseline = "projection_pool")` produces the same output as before. No new errors or warnings from the `pool_baseline` check.

2. **Invalid `pool_baseline` — abort before any other work.** `sgp(..., pool_baseline = <anything other than "projection_pool">)` must abort with class `rotostats_error_invalid_pool_baseline`, message that names both the supplied value and the valid value(s). Must fire before `rate_conversion` validation (i.e., even when `rate_conversion` is also invalid).

3. **Category column absent — fires `rotostats_warning_missing_category_column` ONLY.** When a scored category column is entirely absent from `projections`, the warning class is `rotostats_warning_missing_category_column` and NOT `rotostats_warning_zero_playing_time`.

4. **Zero playing time — fires `rotostats_warning_zero_playing_time` ONLY.** When a player has 0 or NA projected IP (ERA/WHIP scored) or AB (AVG scored), the warning class is `rotostats_warning_zero_playing_time` and NOT `rotostats_warning_missing_category_column`.

5. **No class bleed.** The two warning conditions must not trigger each other's class.

6. **All existing tests pass.** No regression.

---

## 2. Tolerance and Class-Name Policy

- All class comparisons are EXACT string match. Use `class = "..."` in `expect_warning()` or `expect_error()` — do NOT use regex-based message matching for class assertions.
- Use `testthat::expect_error(expr, class = "rotostats_error_invalid_pool_baseline")` or `rlang::expect_error(expr, class = "rotostats_error_invalid_pool_baseline")` — either is acceptable.
- Use `testthat::expect_warning(expr, class = "...")` — do not suppress and inspect the condition object manually unless the `AND NOT` assertion requires it (see §3c and §3d below).
- Numeric comparisons: tolerance `1e-12` for exact arithmetic; `1e-6` for floating-point compound expressions.

---

## 3. New Test Scenarios (add to test-sgp.R, Section 6 or a new Section 8)

### 3a. `pool_baseline` invalid value aborts with correct class

```r
test_that("invalid pool_baseline aborts with rotostats_error_invalid_pool_baseline", {
  cats   <- c("HR")
  denoms <- make_denominators(cats, values = c(HR = 12.0))
  lh     <- make_league_history()
  lc     <- make_league_config()
  proj   <- data.frame(HR = c(20, 30))

  expect_error(
    sgp(proj, denoms, league_history = lh, league_config = lc,
        pool_baseline = "foo"),
    class = "rotostats_error_invalid_pool_baseline"
  )
})
```

Also verify that the two other documented-but-not-implemented values abort under the same class (not `rotostats_error_not_implemented`):

```r
test_that("pool_baseline = 'per_player' aborts with rotostats_error_invalid_pool_baseline", {
  cats   <- c("HR")
  denoms <- make_denominators(cats, values = c(HR = 12.0))
  lh     <- make_league_history()
  lc     <- make_league_config()
  proj   <- data.frame(HR = c(20, 30))

  expect_error(
    sgp(proj, denoms, league_history = lh, league_config = lc,
        pool_baseline = "per_player"),
    class = "rotostats_error_invalid_pool_baseline"
  )
})

test_that("pool_baseline = 'universal_constants' aborts with rotostats_error_invalid_pool_baseline", {
  cats   <- c("HR")
  denoms <- make_denominators(cats, values = c(HR = 12.0))
  lh     <- make_league_history()
  lc     <- make_league_config()
  proj   <- data.frame(HR = c(20, 30))

  expect_error(
    sgp(proj, denoms, league_history = lh, league_config = lc,
        pool_baseline = "universal_constants"),
    class = "rotostats_error_invalid_pool_baseline"
  )
})
```

### 3b. Missing category column fires missing-column class AND NOT zero-playing-time class

```r
test_that("missing category column fires rotostats_warning_missing_category_column not zero_playing_time", {
  cats   <- c("HR", "R")
  denoms <- make_denominators(cats, values = c(HR = 12.0, R = 15.0))
  lh     <- make_league_history()
  lc     <- make_league_config()
  # Projections missing R — condition (a): column absent
  proj   <- data.frame(HR = c(20, 30))

  # Must fire missing_category_column
  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_missing_category_column"
  )

  # Must NOT fire zero_playing_time
  # Collect all warnings and check no zero_playing_time class appears
  w <- tryCatch(
    withCallingHandlers(
      sgp(proj, denoms, league_history = lh, league_config = lc),
      warning = function(cond) {
        if (inherits(cond, "rotostats_warning_zero_playing_time")) {
          stop("rotostats_warning_zero_playing_time should NOT fire for missing column")
        }
        invokeRestart("muffleWarning")
      }
    ),
    error = function(e) fail(conditionMessage(e))
  )
  # If we reach here without failure, zero_playing_time did not fire — test passes
  expect_true(TRUE)
})
```

### 3c. Zero IP fires zero-playing-time class AND NOT missing-column class

```r
test_that("zero IP fires rotostats_warning_zero_playing_time not missing_category_column", {
  cats   <- c("ERA")
  denoms <- make_denominators(cats, values = c(ERA = 0.3))
  lh     <- make_league_history()
  lc     <- make_league_config()
  # ERA column IS present — condition (b): playing time missing
  proj   <- data.frame(ERA = c(3.5, 0.0), WHIP = c(1.2, 0.0), IP = c(200, 0))

  # Must fire zero_playing_time
  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_zero_playing_time"
  )

  # Must NOT fire missing_category_column
  w <- tryCatch(
    withCallingHandlers(
      sgp(proj, denoms, league_history = lh, league_config = lc),
      warning = function(cond) {
        if (inherits(cond, "rotostats_warning_missing_category_column")) {
          stop("rotostats_warning_missing_category_column should NOT fire for zero IP")
        }
        invokeRestart("muffleWarning")
      }
    ),
    error = function(e) fail(conditionMessage(e))
  )
  expect_true(TRUE)
})
```

---

## 4. Existing Tests to Update

### 4a. test-sgp.R line 488 — zero IP ERA test

**Current test (lines 488–500):**
```r
test_that("zero IP player emits rotostats_warning_missing_category_column for ERA", {
  ...
  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_missing_category_column"
  )
})
```

**Required change:** Update class to `"rotostats_warning_zero_playing_time"`.

```r
test_that("zero IP player emits rotostats_warning_zero_playing_time for ERA", {
  cats   <- c("ERA")
  denoms <- make_denominators(cats, values = c(ERA = 0.3))
  lh     <- make_league_history()
  lc     <- make_league_config()
  # One player with IP = 0
  proj   <- data.frame(ERA = c(3.5, 0.0), WHIP = c(1.2, 0.0), IP = c(200, 0))

  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_zero_playing_time"
  )
})
```

### 4b. test-sgp.R lines 517–527 — zero AB AVG test

**Current test (lines 517–527):**
```r
test_that("zero AB player emits warning and gets NA for sgp_AVG", {
  ...
  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_missing_category_column"
  )
  ...
})
```

**Required change:** Update class to `"rotostats_warning_zero_playing_time"`. The second call (`suppressWarnings`) and the `expect_false/expect_true` assertions are unchanged.

```r
test_that("zero AB player emits rotostats_warning_zero_playing_time and gets NA for sgp_AVG", {
  cats   <- c("AVG")
  denoms <- make_denominators(cats, values = c(AVG = 0.003))
  lh     <- make_league_history()
  lc     <- make_league_config()
  proj   <- data.frame(AVG = c(0.270, 0.250), AB = c(550, 0))

  expect_warning(
    sgp(proj, denoms, league_history = lh, league_config = lc),
    class = "rotostats_warning_zero_playing_time"
  )

  result <- suppressWarnings(
    sgp(proj, denoms, league_history = lh, league_config = lc)
  )
  expect_false(is.na(result$sgp_AVG[1L]))
  expect_true(is.na(result$sgp_AVG[2L]))
})
```

---

## 5. Integration Test Updates (test-sgp-integration.R)

Tester must audit each of the three call sites to determine whether the condition under test is (a) column-absent or (b) zero-playing-time, then apply the correct class.

### 5a. Line 520 — TS-10

**Context (lines 514–528):** The test supplies a `projections` frame missing the `SB` column. The warning fires because the SB column is entirely absent — condition (a), column absent.

**Required change:** NONE. Keep `class = "rotostats_warning_missing_category_column"`. This is condition (a) and the existing class is correct after the split.

### 5b. Line 572 — TS-11

**Context (lines 566–583):** The test supplies a player with `IP = 0`. The warning fires because that player has zero projected IP — condition (b), zero playing time.

**Required change:** Change `class = "rotostats_warning_missing_category_column"` to `class = "rotostats_warning_zero_playing_time"` at line 572.

```r
  expect_warning(
    suppressMessages(
      result <- sgp(projections, denominators,
                    league_history = lh, league_config = lc,
                    rate_conversion = "blended_pool")
    ),
    class = "rotostats_warning_zero_playing_time"    # changed from missing_category_column
  )
```

### 5c. Line 625 — TS-12

**Context (lines 619–631):** The test supplies a player with `AB = 0`. The warning fires because that player has zero projected AB — condition (b), zero playing time.

**Required change:** Change `class = "rotostats_warning_missing_category_column"` to `class = "rotostats_warning_zero_playing_time"` at line 625.

```r
  expect_warning(
    suppressMessages(
      result <- sgp(projections, denominators,
                    league_history = lh, league_config = lc,
                    rate_conversion = "blended_pool")
    ),
    class = "rotostats_warning_zero_playing_time"    # changed from missing_category_column
  )
```

---

## 6. Invariants and Property Tests

The following properties must hold after the change:

| Property | How to verify |
|----------|--------------|
| `sgp(..., pool_baseline = "projection_pool")` — same output as before | Run existing tests; all must pass |
| Any invalid `pool_baseline` aborts before `rate_conversion` is checked | Pass `pool_baseline = "foo"` together with `rate_conversion = "invalid_value"` — the error class must be `rotostats_error_invalid_pool_baseline`, not `rotostats_error_invalid_rate_conversion` |
| `rotostats_warning_missing_category_column` does NOT fire on zero-IP/AB rows | §3c above |
| `rotostats_warning_zero_playing_time` does NOT fire on column-absent cases | §3b above |
| Zero-IP player still gets `NA` for `sgp_ERA` / `sgp_WHIP` | Existing test at line 502 — class assertion updates but NA assertion is unchanged |
| Zero-AB player still gets `NA` for `sgp_AVG` | Existing test at lines 529–533 — NA assertion unchanged |
| `total_sgp` is `NA` when any per-category SGP is `NA` | Unchanged by `na.rm = FALSE`; existing tests cover this |

---

## 7. R CMD Check Acceptance

- 0 ERRORs.
- 0 WARNINGs.
- NOTEs: compare the NOTE count on `develop` HEAD vs the feature branch. The feature branch must not introduce any new NOTEs beyond those present on `develop`. Tester must run `devtools::check()` on both branches and diff the NOTE list.
- Suggested command: `devtools::check(error_on = "warning")` — this will surface any ERRORs or WARNINGs as a non-zero exit.

---

## 8. Unit-Test Acceptance

- All existing tests in `tests/testthat/test-sgp.R` and `tests/testthat/test-sgp-integration.R` must pass after the class updates in §4 and §5.
- The three new tests in §3 (plus the two supplementary `pool_baseline` tests for `"per_player"` and `"universal_constants"`) must pass.
- Suggested command: `devtools::test(filter = "sgp")` to run only the sgp test files.

---

## 9. Regression Scenarios (backward compatibility)

| Scenario | Expected outcome |
|----------|-----------------|
| `sgp(..., pool_baseline = "projection_pool")` with valid inputs | Identical output to pre-change; no new warnings or errors |
| `withCallingHandlers(rotostats_warning_missing_category_column = ...)` on a projections frame missing a column | Handler fires as before |
| `withCallingHandlers(rotostats_warning_missing_category_column = ...)` on a projections frame with IP=0 | Handler does NOT fire (class has changed to `rotostats_warning_zero_playing_time`) — this is the documented minor breaking change |
| `withCallingHandlers(rotostats_warning_zero_playing_time = ...)` on a projections frame with IP=0 | Handler fires — new behavior |
