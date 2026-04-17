# Test Spec: `replacement_level()` + `replacement_from_prices()`

Request ID: replacement-2026-04-16
Agent: planner → tester
Date: 2026-04-17
Profile: r-package

This spec is the sole input to the tester pipeline. Tester does not see `spec.md` or `sim-spec.md`. All test scenarios here are self-contained.

---

## 0. Overview

This document defines the complete test plan for:
- `replacement_level()` — per-position replacement-level stat line estimator
- `replacement_from_prices()` — price-based replacement level estimator
- `default_replacement_params` — exported parameter defaults list
- `rate_stat_denominators()` — exported rate-stat lookup helper

Test files:
- `tests/testthat/test-replacement.R` — unit tests for `replacement_level()`, `default_replacement_params`, `rate_stat_denominators()`
- `tests/testthat/test-replacement-integration.R` — integration tests (replacement × sgp iteration loop)
- `tests/testthat/test-replacement-from-prices.R` — unit tests for `replacement_from_prices()` and name normalization

Extend `tests/testthat/helper-test-data.R` with a `make_projections_data()` generator (do not modify existing helpers).

---

## 1. Shared Test Fixtures

Define in `tests/testthat/helper-test-data.R`:

### `make_projections_data(n_hitters = 60, n_sp = 30, n_rp = 20, seed = 42L)`

Returns a data.frame with columns:
- `player_id` (character, unique)
- `player_name` (character)
- `pos_eligibility` (character, pipe-delimited)
- `team` (character, one of a fixed set of MLB abbreviations)
- `league` (character, either `"AL"` or `"NL"`)
- `HR`, `R`, `RBI`, `SB`, `AVG`, `AB` (numeric hitter stats — NA for pitchers)
- `W`, `K`, `SV`, `ERA`, `WHIP`, `IP` (numeric pitcher stats — NA for hitters)
- `role` (character, `"SP"` or `"RP"` for pitchers, NA for hitters)

Hitters: `pos_eligibility` drawn from `c("C", "1B", "2B", "3B", "SS", "OF")` with realistic frequencies. ~15% of hitters are multi-eligible (pipe-delimited).

Pitchers: first `n_sp` rows are SP (IP ~ 160–185, role = "SP"); remaining `n_rp` are RP (IP ~ 55–70, role = "RP").

Use `set.seed(seed)` for reproducibility.

### Standard config fixtures

```r
# 12-team mixed league, standard 5x5
cfg_mixed_12 <- league_config(
  n_teams       = 12L,
  roster_slots  = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L, UTIL=1L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories    = c("HR", "R", "RBI", "SB", "AVG", "W", "K", "SV", "ERA", "WHIP"),
  league_type   = "mixed",
  budget        = 260L
)

# 12-team AL-only league (thin pools for C and SS)
cfg_al_12 <- league_config(
  n_teams       = 12L,
  roster_slots  = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L, DH=1L, UTIL=1L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories    = c("HR", "R", "RBI", "SB", "AVG", "W", "K", "SV", "ERA", "WHIP"),
  league_type   = "AL",
  budget        = 260L
)
```

---

## 2. Output Contract Tests

### TS-01: Basic return structure

**Input:** `make_projections_data()` with `n_hitters=80, n_sp=40, n_rp=25`; `cfg_mixed_12`

**Expected:**

```r
result <- replacement_level(proj, config = cfg_mixed_12)

# Top-level list elements present
expect_named(result, c("replacement_stats", "positional_adjustments",
                        "cliff_metric", "two_way_players",
                        "pool_diagnostics", "method", "params"),
             ignore.order = TRUE)

# method
expect_equal(result$method, "boundary_band")

# replacement_stats structure
expect_s3_class(result$replacement_stats, "data.frame")
expect_true("position" %in% names(result$replacement_stats))
expect_true("n_band_players" %in% names(result$replacement_stats))
expect_true("cliff_detected" %in% names(result$replacement_stats))
# pitcher rows have IP
sp_row <- result$replacement_stats[result$replacement_stats$position == "SP", ]
expect_true("IP" %in% names(result$replacement_stats))
expect_true(!is.na(sp_row$IP))
```

### TS-02: Output attributes present

**Input:** same as TS-01

**Expected:**

```r
expect_equal(attr(result, "stat_units"), "raw_projected")
expect_s3_class(attr(result, "config"), "league_config")
expect_true(is.data.frame(attr(result, "projections")))
expect_true(is.character(attr(result, "position_assignments")))
expect_true(!is.null(names(attr(result, "position_assignments"))))
expect_true(is.logical(attr(result, "converged")))
expect_true(is.integer(attr(result, "iterations")) || is.numeric(attr(result, "iterations")))
expect_true(is.numeric(attr(result, "delta")))
```

### TS-03: `normalize_to_season = TRUE` sets stat_units

**Input:** same projections; `normalize_to_season = TRUE`

**Expected:**

```r
result2 <- replacement_level(proj, config = cfg_mixed_12, normalize_to_season = TRUE)
expect_equal(attr(result2, "stat_units"), "full_season_normalized")
```

### TS-04: `params` list keys

**Input:** same as TS-01

**Expected:**

```r
expect_named(result$params,
             c("converged", "iterations", "delta", "n_teams", "roster_slots",
               "band_width", "cliff_threshold", "sort_by", "stat_units",
               "catcher_adjustment_method", "method"),
             ignore.order = TRUE)
expect_equal(result$params$n_teams, 12L)
expect_equal(result$params$band_width, 3L)  # default K
```

---

## 3. Boundary Band Tests

### TS-05: Counting stat replacement = arithmetic mean of band

**Setup:** Construct a minimal projections data frame for a single position (1B) with exactly 20 players (12 teams × 1 slot = boundary at player 12). Players sorted by HR descending: HR = 40, 38, 36, ..., 4, 2 (step -2). K = 3 → K_eff = min(3, floor(12/4)) = 3 → band players 9 through 15.

```r
proj_1b <- data.frame(
  player_id     = paste0("P", 1:20),
  player_name   = paste0("Player", 1:20),
  pos_eligibility = rep("1B", 20),
  team          = rep("NYY", 20),
  league        = rep("AL", 20),
  HR            = seq(40, 2, by = -2),  # 40, 38, ..., 2
  R             = rep(70, 20),
  RBI           = rep(80, 20),
  SB            = rep(5, 20),
  AVG           = rep(0.260, 20),
  AB            = rep(450, 20)
)
cfg_1b <- league_config(
  n_teams = 12L,
  roster_slots = c(`1B` = 1L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories = c("HR", "R", "RBI", "SB", "AVG"),
  league_type = "AL"
)
result <- replacement_level(proj_1b, config = cfg_1b)
repl_hr <- result$replacement_stats[result$replacement_stats$position == "1B", "HR"]
```

**Expected:** Band players 9–15 have HR = 24, 22, 20, 18, 16, 14, 12. Mean = (24+22+20+18+16+14+12)/7 = 18.0. No cliff (all players equally spaced). Assert `abs(repl_hr - 18.0) < 1e-9`.

### TS-06: AVG replacement = sum(H)/sum(AB), not mean(AVG)

**Setup:** Band of 5 players with varying AB and AVG. Set K=2, 10 players, boundary at 6 (12 teams × 1 slot not practical here — set up a simple 5-player band directly via `band_width = 2` and a small config).

```r
# 5 teams, 1 SS slot → boundary at 5; K=2; K_eff = min(2, floor(5/4)) = min(2,1) = 1
# band = players 4,5,6 (clamped to 5 for lower half = just player 5+1=6 would be outside)
# Simpler: use 20 teams, 1 SS each → boundary at 20; K=2; K_eff=min(2,5)=2; band 18-22
# Set up 25 SS-eligible players:
band_players <- data.frame(
  AB  = c(350, 420, 480),   # players 18, 19, 20 (boundary), 21, 22 after sorting
  AVG = c(0.240, 0.250, 0.260)
)
# H per player = AVG * AB = 84, 105, 124.8
# sum(H) / sum(AB) = (84 + 105 + 124.8) / (350 + 420 + 480) = 313.8 / 1250 = 0.25104
# mean(AVG) = (0.240 + 0.250 + 0.260) / 3 = 0.250 -- these differ
```

**Expected:** `abs(repl_avg - 0.25104) < 1e-4`. The result must NOT equal 0.250 (the arithmetic mean of AVG values). This confirms `sum(H)/sum(AB)` aggregation.

### TS-07: ERA and WHIP replacement = IP-weighted mean

**Setup:** SP band of 3 pitchers with different IP, ERA, WHIP:

```r
# IP = c(180, 170, 155); ERA = c(3.20, 3.60, 4.00); WHIP = c(1.10, 1.20, 1.30)
# Weighted ERA = (3.20*180 + 3.60*170 + 4.00*155) / (180+170+155)
#              = (576 + 612 + 620) / 505
#              = 1808 / 505 = 3.580198...
# Weighted WHIP = (1.10*180 + 1.20*170 + 1.30*155) / 505
#               = (198 + 204 + 201.5) / 505
#               = 603.5 / 505 = 1.19505...
```

**Expected:** `abs(repl_era - 1808/505) < 1e-6`. `abs(repl_whip - 603.5/505) < 1e-6`.

### TS-08: Dynamic K cap prevents band > 25% of pool

**Setup:** 12-team AL, SS = 1 slot → `n_rostered_pos = 12`. K = 3. Expected `K_eff = min(3, floor(12/4)) = 3`. Band = 2*3+1 = 7 players out of 12 = 58%. K_eff is still 3 here (floor(12/4) = 3 = K).

Now test with a thinner pool: configure 5 teams, 1 SS slot → `n_rostered_pos = 5`. K = 3. `K_eff = min(3, floor(5/4)) = min(3, 1) = 1`. Band = 3 players.

```r
cfg_thin <- league_config(
  n_teams = 5L,
  roster_slots = c(SS = 1L),
  pitcher_slots = c(SP=3L, RP=2L),
  categories = c("HR", "R", "RBI", "SB", "AVG"),
  league_type = "AL"
)
proj_thin <- make_projections_data(n_hitters = 15, n_sp=5, n_rp=4, seed=99L)
# filter to SS-eligible only — ensure at least 5 players
result_thin <- replacement_level(proj_thin, config = cfg_thin)
```

**Expected:**

```r
# n_band_players for SS should be <= 3 (2*K_eff+1 = 3)
ss_row <- result_thin$replacement_stats[result_thin$replacement_stats$position == "SS", ]
expect_lte(ss_row$n_band_players, 3L)
```

---

## 4. Cliff Detection Tests

### TS-09: Cliff detection skipped when lower half < `cliff_min_n`

**Setup:** Pool where `K_eff = 2` and lower half has only 2 players (< default `cliff_min_n = 4`). This occurs when K_eff is reduced by the dynamic cap (thin pool).

**Expected:**

```r
result <- replacement_level(proj, config = cfg_thin, band_width = 2L)
cliff_row <- result$cliff_metric[result$cliff_metric$position == "SS", ]
expect_false(cliff_row$cliff_detected)
```

### TS-10: Cliff detected in lower half (MAD method)

**Setup:** 12-team league, 1 SS slot, 25 players. Players 1–12 have SS (projected HR from 30 down to 8 in uniform steps ~2.0 HR apart). Then a cliff: player 13 jumps from 8 to 1 HR (gap = 7). Players 14–15 have HR = 0.5, 0.3. K = 3 → band = players 9–15. Lower half = players 13, 14, 15. Gap between player 12 (HR=8) and player 13 (HR=1) = 7.

```r
proj_cliff <- data.frame(
  player_id     = paste0("SS", 1:25),
  player_name   = paste0("SS_Player", 1:25),
  pos_eligibility = rep("SS", 25),
  team          = rep("BOS", 25),
  league        = rep("AL", 25),
  HR = c(30, 28, 26, 24, 22, 20, 18, 16, 14, 12, 10, 8,  # 12 rostered
          1, 0.5, 0.3,                                      # cliff below boundary
          rep(0, 10)),
  R   = rep(50, 25),
  RBI = rep(60, 25),
  SB  = rep(3, 25),
  AVG = rep(0.250, 25),
  AB  = rep(300, 25)
)
result_cliff <- replacement_level(
  proj_cliff,
  config     = cfg_al_12,
  cliff_method    = "mad",
  cliff_threshold = 1.5
)
```

**Expected:**

```r
ss_cliff <- result_cliff$cliff_metric[result_cliff$cliff_metric$position == "SS", ]
expect_true(ss_cliff$cliff_detected)
# Band was truncated — n_band_players < 7 (the default 2*3+1)
expect_lt(ss_cliff$n_band_players, 7L)
```

### TS-11: Cliff detection applies only to lower half

**Setup:** Introduce a large gap ABOVE the boundary (between players 8 and 9) rather than below it.

**Expected:**

```r
# Cliff in upper half is NOT detected — cliff detection only applies to the lower half
expect_false(cliff_row_upper$cliff_detected)
```

---

## 5. Zero-Sum Assertion Tests

### TS-12: Zero-sum holds for default `catcher_adjustment_method = "split_pool"`

**Setup:** `make_projections_data(n_hitters=100, n_sp=40, n_rp=25, seed=42L)` with `cfg_mixed_12`.

**Expected:**

```r
result <- replacement_level(proj, config = cfg_mixed_12)
pa <- result$positional_adjustments
rs <- cfg_mixed_12$roster_slots

# zero_sum_positions = setdiff(c("C","1B","2B","3B","SS","OF","DH"), "C")
#                    = c("1B","2B","3B","SS","OF")
# DH absent in this mixed config (no DH slot here), so only non-C primary slots
zs_pos <- setdiff(c("C","1B","2B","3B","SS","OF"), "C")
zs_pos <- intersect(zs_pos, names(pa))
zs_check <- sum(rs[zs_pos] * pa[zs_pos])
expect_lt(abs(zs_check), 1e-6)
```

### TS-13: Zero-sum holds for `catcher_adjustment_method = "positional_default"`

Same as TS-12 but with `catcher_adjustment_method = "positional_default"`.

**Expected:**

```r
result_pd <- replacement_level(proj, config = cfg_mixed_12,
                               catcher_adjustment_method = "positional_default")
pa_pd <- result_pd$positional_adjustments
# zero_sum_positions includes "C" here
zs_pos_pd <- intersect(c("C","1B","2B","3B","SS","OF"), names(pa_pd))
zs_check_pd <- sum(rs[zs_pos_pd] * pa_pd[zs_pos_pd])
expect_lt(abs(zs_check_pd), 1e-6)
```

### TS-14: Zero-sum holds for `catcher_adjustment_method = "none"`

Same assertion with `catcher_adjustment_method = "none"`.

### TS-15: Zero-sum holds for `catcher_adjustment_method = "partial_offset"`

Same assertion with `catcher_adjustment_method = "partial_offset"`.

### TS-16: Scarcity premium direction — C and SS positive, OF at or below zero

**Setup:** Mixed league with realistic projections (use `make_projections_data` with enough depth). Assumes C and SS are genuinely scarce relative to OF and 1B.

```r
result <- replacement_level(proj, config = cfg_mixed_12)
pa <- result$positional_adjustments
expect_gt(pa["C"], 0)
expect_gt(pa["SS"], 0)
expect_lte(pa["OF"], 0)
```

Note: This test may be sensitive to the exact projection data. Use a sufficiently large, realistic synthetic pool (n_hitters >= 100).

---

## 6. SP/RP Separation and Role Inference Tests

### TS-17: SP and RP always produce separate replacement rows

**Expected:**

```r
result <- replacement_level(proj, config = cfg_mixed_12)
positions <- result$replacement_stats$position
expect_true("SP" %in% positions)
expect_true("RP" %in% positions)
# SP and RP rows are independent
sp_era <- result$replacement_stats[result$replacement_stats$position == "SP", "ERA"]
rp_era <- result$replacement_stats[result$replacement_stats$position == "RP", "ERA"]
expect_false(isTRUE(all.equal(sp_era, rp_era)))  # SP and RP should differ
```

### TS-18: SP/RP inferred from IP when `role` column absent

**Setup:** Remove the `role` column from `make_projections_data()` output. Use `sp_ip_threshold = 100` (default).

**Expected:**

```r
proj_no_role <- proj[, setdiff(names(proj), "role")]
result_inferred <- replacement_level(proj_no_role, config = cfg_mixed_12)
# Should still produce SP and RP rows without error
expect_true("SP" %in% result_inferred$replacement_stats$position)
expect_true("RP" %in% result_inferred$replacement_stats$position)
```

### TS-19: Explicit `role` column used directly when present

**Setup:** Assign `role = "SP"` to pitchers with IP < 100 (artificially override inference).

**Expected:** The function uses the explicit `role` column, not IP-based inference. No warning emitted.

### TS-20: Swingman flag set for pitchers with 80 <= IP <= 120

**Setup:** Include at least one pitcher with IP = 95 (within swingman range) and one with IP = 150 (not swingman).

**Expected:**

```r
# cliff_metric$swingman should be TRUE for any position where a swingman is in the band
# Position assignments are checked, not individual player flags in cliff_metric
swingman_rows <- result$cliff_metric[result$cliff_metric$swingman == TRUE, ]
# At least one swingman-flagged position exists (for SP band if the swingman is near the boundary)
# This test is data-dependent — construct projections so a swingman sits near the SP boundary
```

### TS-21: Pitcher `replacement_stats` always includes IP column

**Expected:**

```r
result <- replacement_level(proj, config = cfg_mixed_12)
# IP present even when ERA and WHIP are in categories (already needed), but also
# verify specifically that the column is there
expect_true("IP" %in% names(result$replacement_stats))
sp_ip <- result$replacement_stats[result$replacement_stats$position == "SP", "IP"]
expect_true(!is.na(sp_ip))
expect_gt(sp_ip, 0)
```

---

## 7. Invalid Pairings and Error Tests

### TS-22: `sort_by = "sgp"` without `sgp_denominators` → error

```r
expect_error(
  replacement_level(proj, config = cfg_mixed_12, sort_by = "sgp"),
  class = "rotostats_error_missing_sgp_denominators"
)
```

### TS-23: `boundary_rate_method = "sgp_pool"` × `rate_conversion = "fixed_baseline"` → error

**Setup:** Create a mock `sgp_denominators` object with `attr(., "rate_conversion") = "fixed_baseline"`.

```r
fake_denoms <- structure(
  list(denominators = c(ERA = 0.5, WHIP = 0.05)),
  class = c("sgp_denominators", "list"),
  rate_conversion = "fixed_baseline"
)
expect_error(
  replacement_level(proj, config = cfg_mixed_12,
                    boundary_rate_method = "sgp_pool",
                    sgp_denominators = fake_denoms),
  class = "rotostats_error_rate_method_mismatch"
)
```

### TS-24: Error message for rate_method_mismatch names both methods

```r
err <- tryCatch(
  replacement_level(proj, config = cfg_mixed_12,
                    boundary_rate_method = "sgp_pool",
                    sgp_denominators = fake_denoms),
  error = function(e) e
)
expect_match(conditionMessage(err), "fixed_baseline")
expect_match(conditionMessage(err), "sgp_pool")
```

### TS-25: `projections` missing required column → error

```r
proj_bad <- proj[, setdiff(names(proj), "player_id")]
expect_error(
  replacement_level(proj_bad, config = cfg_mixed_12),
  class = "rotostats_error_missing_column"
)
```

### TS-26: Wrong column type → error

```r
proj_bad_type <- proj
proj_bad_type$HR <- as.character(proj_bad_type$HR)
expect_error(
  replacement_level(proj_bad_type, config = cfg_mixed_12),
  class = "rotostats_error_wrong_column_type"
)
```

### TS-27: Unknown rate stat → error

```r
cfg_custom_cat <- league_config(
  n_teams = 12L,
  roster_slots = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories = c("HR", "BABIP"),   # BABIP not in RATE_STAT_DENOMINATORS
  league_type = "mixed"
)
proj_babip <- proj
proj_babip$BABIP <- 0.300
expect_error(
  replacement_level(proj_babip, config = cfg_custom_cat),
  class = "rotostats_error_unknown_rate_stat"
)
```

### TS-28: Pool too small → error

**Setup:** `n_teams = 15`, `roster_slots = c(SS = 2)` → boundary at 30. Provide only 20 SS-eligible players.

```r
expect_error(
  replacement_level(thin_proj, config = cfg_deep_ss),
  class = "rotostats_error_pool_too_small"
)
```

### TS-29: `positional_adjustment_method = "posblend"` without `pos_weight` → error

```r
expect_error(
  replacement_level(proj, config = cfg_mixed_12,
                    positional_adjustment_method = "posblend"),
  class = "rotostats_error_missing_pos_weight"
)
```

### TS-30: `seed_method = "historical_priors"` without `league_history` → error

```r
expect_error(
  replacement_level(proj, config = cfg_mixed_12,
                    seed_method = "historical_priors"),
  class = "rotostats_error_missing_league_history"
)
```

---

## 8. Convergence Tests

### TS-31: `converged = TRUE` on typical inputs (no multi-pos churn)

**Setup:** All players are single-position eligible (no multi-pos). `sort_by = "zscore"`. No SGP iteration needed.

**Expected:**

```r
result <- replacement_level(proj_single_pos, config = cfg_mixed_12,
                             multi_pos = "primary")
expect_true(attr(result, "converged"))
# Should converge in 1 pass since no position changes possible
expect_equal(attr(result, "iterations"), 1L)
```

### TS-32: `converged = FALSE` + warning at `max_iter`

**Setup:** Construct a deliberately non-convergent scenario: a pool where two multi-eligible players perpetually flip positions (not physically feasible, so instead set `max_iter = 1` to force the cap to trigger).

```r
expect_warning(
  result_nc <- replacement_level(proj, config = cfg_mixed_12,
                                  max_iter = 1L),
  class = "rotostats_warning_convergence_not_reached"
)
expect_false(attr(result_nc, "converged"))
expect_equal(attr(result_nc, "iterations"), 1L)
```

### TS-33: Convergence warning fires even when `verbose = FALSE`

```r
# Warning must not be suppressed by verbose = FALSE
withCallingHandlers(
  replacement_level(proj, config = cfg_mixed_12, max_iter = 1L, verbose = FALSE),
  rotostats_warning_convergence_not_reached = function(w) {
    expect_true(TRUE)   # warning fired
    invokeRestart("muffleWarning")
  }
)
```

---

## 9. `default_replacement_params` Tests

### TS-34: Exported list has correct keys and defaults

```r
expect_named(
  default_replacement_params,
  c("band_width_K", "cliff_threshold", "cliff_min_n", "sp_ip_threshold",
    "sp_rp_split_default", "ip_ab_divergence_tol", "calibration_min_n",
    "convergence_eps", "convergence_max_iter"),
  ignore.order = TRUE
)
expect_equal(default_replacement_params$band_width_K, 3L)
expect_equal(default_replacement_params$cliff_threshold, 1.5)
expect_equal(default_replacement_params$cliff_min_n, 4L)
expect_equal(default_replacement_params$sp_ip_threshold, 100)
expect_equal(default_replacement_params$sp_rp_split_default, c(SP = 0.60, RP = 0.40))
expect_equal(default_replacement_params$ip_ab_divergence_tol, 0.15)
expect_equal(default_replacement_params$calibration_min_n, 15L)
expect_equal(default_replacement_params$convergence_eps, 0.01)
expect_equal(default_replacement_params$convergence_max_iter, 25L)
```

### TS-35: `replacement_params` overrides take effect

```r
result_narrow <- replacement_level(proj, config = cfg_mixed_12,
                                    replacement_params = list(band_width_K = 1L))
expect_equal(result_narrow$params$band_width, 1L)
# n_band_players should be <= 3 (2*1+1)
expect_lte(max(result_narrow$replacement_stats$n_band_players), 3L)
```

### TS-36: Top-level `band_width` wins over `replacement_params`

```r
result_conflict <- replacement_level(proj, config = cfg_mixed_12,
                                      band_width = 2L,
                                      replacement_params = list(band_width_K = 5L))
expect_equal(result_conflict$params$band_width, 2L)
```

---

## 10. `rate_stat_denominators()` Tests

### TS-37: Returns named character vector

```r
rsd <- rate_stat_denominators()
expect_true(is.character(rsd))
expect_true(!is.null(names(rsd)))
```

### TS-38: Contains required mappings

```r
rsd <- rate_stat_denominators()
expect_equal(rsd[["AVG"]],  "AB")
expect_equal(rsd[["ERA"]],  "IP")
expect_equal(rsd[["WHIP"]], "IP")
expect_equal(rsd[["OBP"]],  "PA")
expect_equal(rsd[["SLG"]],  "AB")
```

### TS-39: Custom `rate_denominators` extends the lookup

**Setup:** Provide `rate_denominators = c(BABIP = "AB")` and include a `BABIP` column in projections.

**Expected:** No `rotostats_error_unknown_rate_stat` error; BABIP is treated as a rate stat with AB denominator.

---

## 11. `replacement_from_prices()` Tests

File: `tests/testthat/test-replacement-from-prices.R`

### `make_prices_data(n_players = 50, seed = 42L)`

Helper that generates a minimal `prices` data frame with columns: `year`, `player_id`, `player_name`, `price`, `pos_eligibility`, `is_keeper` (logical).

### TS-40: Basic return structure

```r
result_prices <- replacement_from_prices(
  prices        = make_prices_data(n_players = 60, seed = 7L),
  n_teams       = 12L,
  roster_slots  = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L),
  categories    = c("HR", "R", "RBI", "SB", "AVG")
)
expect_named(result_prices,
             c("replacement_stats", "positional_adjustments",
               "cliff_metric", "two_way_players",
               "pool_diagnostics", "method", "params"),
             ignore.order = TRUE)
expect_equal(result_prices$params$method, "prices")
```

### TS-41: `is_keeper = TRUE` rows excluded exactly

**Setup:** 30 $1 players; 5 of them have `is_keeper = TRUE` with above-replacement stats.

**Expected:** The 5 keeper players are excluded. Trimming is not applied (because `is_keeper` column is present).

```r
# Create prices where 5 high-stat players are keepers
prices_keeper <- make_prices_data_with_keepers(n_regular = 25, n_keeper = 5, seed = 11L)
result_k <- replacement_from_prices(prices_keeper, n_teams=12L, ...)
# Verify that the 5 keeper players do not appear in the effective $1 pool
# (indirect: check that replacement stat is not inflated by the keepers' high stats)
```

### TS-42: Trim method `"iqr"` removes overperformers when `is_keeper` absent

```r
prices_no_keeper <- make_prices_data(seed=42L)
prices_no_keeper$is_keeper <- NULL   # remove is_keeper column
result_trim <- replacement_from_prices(prices_no_keeper, n_teams=12L, ..., trim_method="iqr")
# Should complete without error; positional_adjustments present
expect_false(is.null(result_trim$positional_adjustments))
```

### TS-43: Returns `NULL` when < `calibration_min_n` players after trimming

```r
prices_sparse <- make_prices_data(n_players = 5, seed=99L)   # very few $1 players
expect_warning(
  result_null <- replacement_from_prices(prices_sparse, n_teams=12L, ...,
                                          calibration_min_n = 15L),
  class = "rotostats_warning_calibration_suppressed"
)
expect_null(result_null)
```

### TS-44: `method = "prices"` in returned `params`

```r
result_p <- replacement_from_prices(make_prices_data(), n_teams=12L, ...)
expect_equal(result_p$params$method, "prices")
```

---

## 12. Name Normalization Tests

File: `tests/testthat/test-replacement-from-prices.R` (or a shared helper test file)

These tests validate the Unicode NFD → diacritic strip → lowercase → alnum → squish normalization pipeline used when `player_id` is absent in `$prices`.

### TS-45: ASCII names unchanged

```r
normalize_name_fn <- # access via internal testing mechanism or test via replacement_from_prices behavior
expect_equal(normalize_name_fn("Mike Trout"), "mike trout")
expect_equal(normalize_name_fn("Jose Ramirez"), "jose ramirez")
```

### TS-46: Diacritics stripped

```r
expect_equal(normalize_name_fn("José Ramírez"), "jose ramirez")
expect_equal(normalize_name_fn("Yoán Moncada"), "yoan moncada")
expect_equal(normalize_name_fn("Nomar Garciaparra"), "nomar garciaparra")
```

### TS-47: Special characters removed

```r
expect_equal(normalize_name_fn("A.J. Pollock"), "aj pollock")
expect_equal(normalize_name_fn("J.D. Martinez"), "jd martinez")
```

### TS-48: Multiple spaces collapsed

```r
expect_equal(normalize_name_fn("Mike  Trout"), "mike trout")
```

### TS-49: Missing names produce `rotostats_warning_name_match_failure`

**Setup:** Include a player in projections whose normalized name does not match any name in `$prices`.

```r
expect_warning(
  replacement_from_prices(prices_unmatched, ..., verbose = TRUE),
  class = "rotostats_warning_name_match_failure"
)
# Warning is suppressed when verbose = FALSE
expect_no_warning(
  replacement_from_prices(prices_unmatched, ..., verbose = FALSE)
)
```

---

## 13. Integration Tests

File: `tests/testthat/test-replacement-integration.R`

### TS-50: `sort_by = "sgp"` iteration with valid denominators converges

**Setup:** Create a valid `sgp_denominators` object using existing test helpers. Confirm the loop converges.

```r
# Requires: sgp_denominators from existing test helpers
denoms <- make_test_sgp_denominators()  # uses existing helper-test-data.R patterns
result_sgp <- replacement_level(
  proj,
  config           = cfg_mixed_12,
  sort_by          = "sgp",
  sgp_denominators = denoms
)
expect_true(attr(result_sgp, "converged"))
expect_gte(attr(result_sgp, "iterations"), 1L)
```

### TS-51: `sort_by = "sgp"` produces different replacement stats than `sort_by = "zscore"`

```r
result_z   <- replacement_level(proj, config = cfg_mixed_12, sort_by = "zscore")
result_sgp <- replacement_level(proj, config = cfg_mixed_12, sort_by = "sgp",
                                  sgp_denominators = denoms)
# The two methods should produce different (but correlated) results
# Allow for small differences due to ranking differences
z_sp_era   <- result_z$replacement_stats[result_z$replacement_stats$position == "SP", "ERA"]
sgp_sp_era <- result_sgp$replacement_stats[result_sgp$replacement_stats$position == "SP", "ERA"]
# Not identical (would indicate one method is not using the denominator)
expect_false(isTRUE(all.equal(z_sp_era, sgp_sp_era, tolerance = 0)))
```

### TS-52: `position_assignments` on second call updates pool membership

**Setup:** Simulate a two-pass call where `position_assignments` from the first call is fed into a second call.

```r
result1 <- replacement_level(proj, config = cfg_mixed_12, multi_pos = "highest_par")
pa1 <- attr(result1, "position_assignments")

# Supply the assignments from pass 1 as position_assignments on a second call
result2 <- replacement_level(proj, config = cfg_mixed_12,
                               multi_pos = "highest_par",
                               position_assignments = pa1)
# Both results should be valid; result2 may differ slightly from result1
expect_named(result2, names(result1), ignore.order = TRUE)
```

### TS-53: `multi_pos = "primary"` ignores `position_assignments`

```r
result_prim <- replacement_level(proj, config = cfg_mixed_12,
                                   multi_pos = "primary",
                                   position_assignments = pa1)  # pa1 is ignored
# Assignment uses first position in pos_eligibility, not pa1
# Verify determinism: calling twice with different position_assignments produces same result
result_prim2 <- replacement_level(proj, config = cfg_mixed_12,
                                    multi_pos = "primary",
                                    position_assignments = NULL)
expect_equal(result_prim$replacement_stats, result_prim2$replacement_stats)
```

---

## 14. Property-Based Invariants

### TS-54: `replacement_stats` position column matches config positions

```r
result <- replacement_level(proj, config = cfg_mixed_12)
valid_positions <- c(names(cfg_mixed_12$roster_slots), "SP", "RP")
expect_true(all(result$replacement_stats$position %in% valid_positions))
```

### TS-55: `n_band_players >= 1` for all positions

```r
expect_true(all(result$replacement_stats$n_band_players >= 1L))
```

### TS-56: All stat columns in `replacement_stats` are numeric

```r
stat_cols <- setdiff(names(result$replacement_stats),
                     c("position", "n_band_players", "cliff_detected"))
for (col in stat_cols) {
  expect_true(is.numeric(result$replacement_stats[[col]]),
              info = paste("column", col, "is not numeric"))
}
```

### TS-57: `position_assignments` covers all players with valid positions

```r
pa <- attr(result, "position_assignments")
expect_true(is.character(pa))
expect_true(!is.null(names(pa)))
valid_pos <- c(names(cfg_mixed_12$roster_slots), "SP", "RP")
expect_true(all(pa %in% valid_pos))
```

### TS-58: `stat_units` attribute is always set

```r
expect_true(attr(result, "stat_units") %in% c("raw_projected", "full_season_normalized"))
```

---

## 15. Validation Commands

Run in order before opening a PR:

```r
# 1. Regenerate docs (required after any roxygen2 change)
devtools::document()

# 2. Full check — must produce zero errors, zero warnings, zero notes
#    (NSE "no visible binding" notes must be suppressed via utils::globalVariables())
devtools::check()

# 3. All tests pass
devtools::test()
```

Expected outputs:
- `devtools::check()`: `0 errors | 0 warnings | 0 notes`
- `devtools::test()`: all tests pass; no skips (unless explicitly marked)

---

## 16. Simulation Validation (from Monte Carlo study)

Tester validates the simulation results produced by the simulator by asserting the following acceptance criteria. The simulator outputs a `simulation.md` or structured result table in the run directory; tester reads that and asserts:

### SV-01: Boundary-band stability (K=3 vs K=1)

- Across 500 simulation replications with projection perturbation (`sigma_proj = 0.10 * mean_stat`):
  - Year-over-year variance of the band-mean replacement HR for 1B with K=3 must be strictly less than year-over-year variance with K=1
  - Formally: `var(repl_HR_K3) < var(repl_HR_K1)` across replications

### SV-02: Zero-sum invariant holds under all four catcher methods

- Across all four `catcher_adjustment_method` values and all simulation scenarios:
  - Maximum observed `abs(sum(roster_slots * scarcity_premium))` < 1e-6 in every replication
  - Zero violations across all 500 × 4 = 2000 replication-method combinations

### SV-03: Iteration loop convergence on multi-eligible pools

- Convergence rate (`converged = TRUE`) >= 99% across 500 replications on the multi-eligible DGP
- When convergence achieved: median iterations <= 5
- When convergence fails: `rotostats_warning_convergence_not_reached` was emitted (check via `withCallingHandlers`)

### SV-04: Dynamic K cap in thin AL-only pools

- For 12-team AL SS (`n_rostered_pos = 12`): `K_eff = 3` confirmed in all replications (K=3 default, floor(12/4)=3)
- For 5-team AL C (`n_rostered_pos = 5`): `K_eff = 1` confirmed (`min(3, floor(5/4)) = 1`) in all replications

### SV-05: Rank invariance — raw_ip method

- A synthetic pitcher of fixed quality (ERA=3.50, WHIP=1.15, IP=175) must rank within ±2 positions of the same rank in 10-team vs 15-team league replications
- Median |rank_10team - rank_15team| <= 2 across 500 replications

Tester asserts these criteria pass from the simulator's output table. If any criterion fails, tester reports a failure, not a skip.
