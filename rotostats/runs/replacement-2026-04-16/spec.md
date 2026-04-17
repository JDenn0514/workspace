# Implementation Spec: `replacement_level()` + `replacement_from_prices()`

Request ID: replacement-2026-04-16
Agent: planner → builder
Date: 2026-04-17
Profile: r-package

This spec is the sole input to the builder pipeline. Builder does not see `test-spec.md` or `sim-spec.md`.

---

## 1. Notation

| Symbol | Type | Description |
|--------|------|-------------|
| `n_teams` | integer | Number of teams in the league (from `config$n_teams`) |
| `roster_slots[pos]` | integer | Number of roster slots at position `pos` (from `config$roster_slots`) |
| `pitcher_slots` | integer or named integer vector | SP/RP slot counts (from `config$pitcher_slots`) |
| `n_rostered_pos` | integer | `n_teams * roster_slots[pos]` — total players rostered at `pos` |
| `K` | integer | Band half-width (default 3) |
| `K_eff` | integer | `min(K, floor(n_rostered_pos / 4))` |
| `b` | integer | Boundary index = `n_rostered_pos` |
| `B` | set | Band players: indices `(b - K_eff)` through `(b + K_eff)` in the sorted pool |
| `B_lower` | set | Lower half of band: indices `(b + 1)` through `(b + K_eff)` |
| `scarcity_premium[pos]` | numeric | `global_replacement - replacement[pos]` |
| `zero_sum_positions` | character vector | Position set over which zero-sum is enforced |
| `tol` | numeric | Convergence tolerance (default 0.01 SGP units) |
| `max_iter` | integer | Maximum iteration passes (default 25) |
| `PRIMARY_HITTER_SLOTS` | character vector | `c("C", "1B", "2B", "3B", "SS", "OF", "DH")` — combo slots excluded |

---

## 2. File Layout

```
R/replacement.R            — replacement_level(), replacement_from_prices()
R/replacement_internal.R   — all internal helpers (see §7)
R/replacement_params.R     — default_replacement_params (exported list), rate_stat_denominators() (exported fn)
```

No code from `R/replacement.R` may be duplicated in `R/replacement_internal.R`. The three mandated shared helpers (`format_replacement_output()`, `compute_positional_adjustments()`, `assert_replacement_output_contract()`) live exclusively in `R/replacement_internal.R` and are called by both exported functions.

---

## 3. Exported Function Signatures

### 3.1 `replacement_level()`

```r
replacement_level <- function(
  projections,
  config,
  sort_by                      = "zscore",
  sgp_denominators             = NULL,
  boundary_method              = "head_count",
  seed_method                  = "hierarchy",
  positional_adjustment_method = "fvarz",
  pos_weight                   = NULL,
  boundary_rate_method         = "raw_ip",
  cliff_method                 = "mad",
  cliff_gap_ratio_threshold    = 0.375,
  catcher_adjustment_method    = "split_pool",
  sp_ip_threshold              = 100,
  normalize_to_season          = FALSE,
  band_width                   = NULL,
  cliff_threshold              = 1.5,
  rate_denominators            = NULL,
  league_history               = NULL,
  trim_method                  = "iqr",
  multi_pos                    = "highest_par",
  position_assignments         = NULL,
  replacement_params           = list(),
  max_iter                     = 25L,
  tol                          = 0.01,
  verbose                      = FALSE
)
```

All parameters are exactly as listed. No additional parameters. No global state. No hard-coded defaults beyond those named here.

### 3.2 `replacement_from_prices()`

```r
replacement_from_prices <- function(
  prices,
  n_teams,
  roster_slots,
  categories,
  trim_method       = "iqr",
  calibration_min_n = 15L,
  verbose           = FALSE
)
```

Parameters from `replacement_level()` that do NOT appear here: `projections`, `config`, `sort_by`, `band_width`, `cliff_threshold`, `cliff_method`, `sp_ip_threshold`, `boundary_method`, `seed_method`, `normalize_to_season`, `max_iter`, `tol`, `sgp_denominators`, `positional_adjustment_method`, `multi_pos`, `position_assignments`, `boundary_rate_method`, `catcher_adjustment_method`, `pos_weight`, `replacement_params`.

### 3.3 `default_replacement_params` (exported list)

```r
default_replacement_params <- list(
  band_width_K          = 3L,
  cliff_threshold       = 1.5,
  cliff_min_n           = 4L,
  sp_ip_threshold       = 100,
  sp_rp_split_default   = c(SP = 0.60, RP = 0.40),
  ip_ab_divergence_tol  = 0.15,
  calibration_min_n     = 15L,
  convergence_eps       = 0.01,
  convergence_max_iter  = 25L
)
```

Exported as a named list. Users override specific entries via `replacement_params = list(band_width_K = 2L)`.

### 3.4 `rate_stat_denominators()` (exported function)

```r
rate_stat_denominators <- function()
```

No arguments. Returns the built-in `RATE_STAT_DENOMINATORS` named character vector:

```r
RATE_STAT_DENOMINATORS <- c(
  AVG   = "AB",  OBP   = "PA",   SLG  = "AB",   OPS  = "PA",
  ERA   = "IP",  WHIP  = "IP",  "K/9" = "IP", "BB/9" = "IP", "HR/9" = "IP",
  SVHD  = "G",   QS    = "GS",
  "K%"  = "PA", "BB%" = "PA",
  wOBA  = "PA",  xFIP  = "IP",  SIERA = "IP",   FIP  = "IP"
)
```

`RATE_STAT_DENOMINATORS` is a package-internal constant defined in `R/replacement_params.R`. `rate_stat_denominators()` simply returns it. For any rate stat not in the lookup, `replacement_level()` aborts with `rotostats_error_unknown_rate_stat`.

---

## 4. Input Validation

### 4.1 Validation Order for `replacement_level()`

Perform all checkmate assertions before any computation. Call `rlang::caller_env()` to surface errors in the user's frame. Apply in this order:

1. `checkmate::assert_data_frame(projections, min.rows = 1L)`
2. `checkmate::assert_class(config, "league_config")`
3. `checkmate::assert_choice(sort_by, c("zscore", "sgp"))`
4. If `sort_by == "sgp"` and is.null(sgp_denominators): abort `rotostats_error_missing_sgp_denominators`
5. `checkmate::assert_choice(boundary_method, c("head_count", "playing_time"))`
6. `checkmate::assert_choice(seed_method, c("hierarchy", "historical_priors"))`
7. If `seed_method == "historical_priors"` and is.null(league_history): abort `rotostats_error_missing_league_history`
8. `checkmate::assert_choice(positional_adjustment_method, c("fvarz", "sgp", "dollar", "posblend"))`
9. If `positional_adjustment_method == "posblend"` and is.null(pos_weight): abort `rotostats_error_missing_pos_weight`
10. If `positional_adjustment_method == "posblend"` and !is.null(pos_weight): `checkmate::assert_number(pos_weight, lower = 0, upper = 1)`
11. `checkmate::assert_choice(boundary_rate_method, c("raw_ip", "sgp_pool"))`
12. If `boundary_rate_method == "sgp_pool"` and !is.null(sgp_denominators): check `attr(sgp_denominators, "rate_conversion")`; if `"fixed_baseline"` abort `rotostats_error_rate_method_mismatch` (message must name both `"fixed_baseline"` and `"sgp_pool"` and suggest using `rate_conversion = "blended_pool"` in `sgp_denominators()` or switching to `boundary_rate_method = "raw_ip"`)
13. `checkmate::assert_choice(cliff_method, c("mad", "fisher_jenks", "gap_ratio"))`
14. `checkmate::assert_number(cliff_gap_ratio_threshold, lower = 0, upper = 1)` (when `cliff_method == "gap_ratio"`)
15. `checkmate::assert_choice(catcher_adjustment_method, c("split_pool", "positional_default", "partial_offset", "none"))`
16. `checkmate::assert_number(sp_ip_threshold, lower = 0, finite = TRUE)`
17. `checkmate::assert_flag(normalize_to_season)`
18. If !is.null(band_width): `checkmate::assert_int(band_width, lower = 1L)`
19. `checkmate::assert_number(cliff_threshold, lower = 0)`
20. If !is.null(rate_denominators): `checkmate::assert_character(rate_denominators, names = "named")`
21. `checkmate::assert_choice(trim_method, c("iqr", "mad", "kde"))`
22. `checkmate::assert_choice(multi_pos, c("highest_par", "primary", "all", "custom"))`
23. `checkmate::assert_int(max_iter, lower = 1L)`
24. `checkmate::assert_number(tol, lower = 0)`
25. `checkmate::assert_flag(verbose)`
26. `checkmate::assert_list(replacement_params)`

Then validate required columns in `projections`:

27. Normalize `projections` column names to uppercase at entry (silent, matching `sgp()` convention): `names(projections) <- toupper(names(projections))`
28. Build `stats_required`: union of `config$categories` plus `"IP"` (pitchers) and `"AB"` (hitters when AVG/OBP/SLG in categories), plus `c("player_id", "player_name", "pos_eligibility", "league")`
29. For each required column missing from `projections`: abort `rotostats_error_missing_column`
30. For each scored category column present but not numeric: abort `rotostats_error_wrong_column_type`
31. `checkmate::assert_subset(projections$league, c("AL", "NL"))`
32. For each rate stat in `config$categories`: look up in merged lookup (built-in + `rate_denominators`); if not found abort `rotostats_error_unknown_rate_stat`

### 4.2 Validation Order for `replacement_from_prices()`

1. `checkmate::assert_data_frame(prices, min.rows = 1L)`
2. `checkmate::assert_int(n_teams, lower = 1L)`
3. `checkmate::assert_integer(roster_slots, names = "named", lower = 0L)`
4. `checkmate::assert_character(categories, min.len = 1L)`
5. Required price columns: `c("year", "player_name", "price")`; missing → abort `rotostats_error_missing_column`
6. `checkmate::assert_choice(trim_method, c("iqr", "mad", "kde"))`
7. `checkmate::assert_int(calibration_min_n, lower = 1L)`
8. `checkmate::assert_flag(verbose)`

---

## 5. Algorithm: `replacement_level()`

### 5.1 Parameter Resolution

Merge `replacement_params` (user overrides) with `default_replacement_params`:

```r
params <- modifyList(default_replacement_params, replacement_params)
```

Resolve K:

```r
K <- if (!is.null(band_width)) as.integer(band_width) else as.integer(params$band_width_K)
```

Resolve `cliff_threshold` (top-level parameter wins over params list):

```r
cliff_thr <- cliff_threshold   # top-level arg always wins
```

### 5.2 SP/RP Role Inference and Swingman Flagging

**Compute swingman set BEFORE role classification** (impact.md Risk Area #3):

```r
# Identify pitcher rows (anyone with pos_eligibility containing "SP" or "RP")
is_pitcher <- grepl("(SP|RP)", projections$pos_eligibility)

# Compute swingman BEFORE role assignment
swingman_flag <- is_pitcher & !is.na(projections$IP) &
                 projections$IP >= 80 & projections$IP <= 120

# Now assign roles
if ("role" %in% names(projections)) {
  role <- projections$role
} else {
  role <- ifelse(is_pitcher & !is.na(projections$IP) &
                   projections$IP >= params$sp_ip_threshold, "SP", "RP")
}
```

When a swingman's inclusion shifts the band mean ERA/WHIP by > 0.10, emit `cli_inform()` when `verbose = TRUE`. No behavioral change — swingman status is informational only.

### 5.3 Strip Projections Attribute for Loop

Before entering the iteration loop, strip the `projections` attribute from any existing result object to avoid per-iteration copy:

```r
stored_projections <- projections   # save for reattach
```

(The attribute is attached only to the final result, never to loop-internal objects.)

### 5.4 Pass 1 Seed

When `position_assignments = NULL` (first call from `dollar_values()` or direct user call):

- If `seed_method = "hierarchy"`: rank players within their primary position group only (first position in `pos_eligibility`), using scarcity ordering C > SS > 2B > 3B > 1B > OF.
- If `seed_method = "historical_priors"`: use per-position replacement-level z-scores from `league_history$team_season` to seed (requires `league_history`; already validated in §4.1 step 7).

### 5.5 Iteration Loop

The loop body (passes 1 through `max_iter`):

```
pass = 1
converged = FALSE
old_assignments = NULL
old_repl_stats = NULL

repeat {
  # A. Sort players into position pools using current assignments
  #    (on pass 1, use seed from §5.4)

  # B. For each position pos in config$roster_slots:
  #    1. Filter pool to players assigned to pos
  #    2. Sort by composite rank (z-score on pass 1, total_sgp on pass 2+)
  #    3. Compute n_rostered_pos = n_teams * roster_slots[pos]
  #    4. Compute K_eff = min(K, floor(n_rostered_pos / 4))
  #    5. Identify band B = players at sorted positions (n_rostered_pos - K_eff) to
  #                          (n_rostered_pos + K_eff), clamped to pool size
  #    6. Apply cliff detection (§5.6) to lower half B_lower
  #    7. Compute replacement stat line for pos (§5.7)

  # C. Compute global replacement level separately for hitters and pitchers

  # D. Compute positional adjustments (§5.8) via compute_positional_adjustments()

  # E. Enforce zero-sum assertion (§5.9)

  # F. When pass >= 2 and sort_by = "sgp":
  #    Call sgp() internally to get total_sgp per player:
  #      sgp_result <- sgp(
  #        projections   = projections,    # full projections df
  #        denominators  = sgp_denominators,
  #        league_history = league_history, # forwarded if supplied
  #        league_config  = config
  #      )
  #    Attach total_sgp to projections for re-ranking

  # G. When multi_pos = "highest_par":
  #    Reassign multi-eligible players to position where PAR is highest

  # H. Check convergence:
  #    assignments_converged = !is.null(old_assignments) &&
  #                            all(new_assignments == old_assignments)
  #    stats_converged = !is.null(old_repl_stats) &&
  #                      max(abs(new_repl_stats - old_repl_stats), na.rm = TRUE) < tol
  #    if (assignments_converged && stats_converged) { converged = TRUE; break }
  #    if (pass >= max_iter) { break }

  old_assignments = new_assignments
  old_repl_stats  = new_repl_stats
  pass = pass + 1
}
```

If `pass >= max_iter` and not converged: emit `cli_warn(..., class = "rotostats_warning_convergence_not_reached")` unconditionally (not gated by `verbose`). The warning must fire even when `verbose = FALSE`, following `lme4` convention.

### 5.6 Cliff Detection

Applied to `B_lower` (the lower half of the band: players beyond the boundary) only.

**Guard:** Skip cliff detection when `length(B_lower) < params$cliff_min_n`. Set `cliff_detected = FALSE`.

**Method dispatch:**

**`"mad"` (default):**

```r
band_values <- replacement_stat_values_for_band   # one stat at a time
mad_val     <- stats::mad(band_values, constant = 1.4826)
gaps        <- diff(sort(B_lower_values))          # adjacent gaps in lower half
cliff_pos   <- which(gaps >= cliff_thr * mad_val * 1.4826)
if (length(cliff_pos) > 0L) {
  j             <- min(cliff_pos)          # first cliff in lower half
  cliff_detected <- TRUE
  # truncate: use only band players from (b - K_eff) through (b + j - 1)
} else {
  cliff_detected <- FALSE
}
```

**`"fisher_jenks"`:**

Requires `classInt` package. Use `requireNamespace("classInt", quietly = TRUE)`; if not available, abort with informative message directing user to install `classInt`.

```r
breaks <- classInt::classIntervals(band_values, n = 2, style = "fisher")$brks
split_point <- breaks[2]   # the Fisher-Jenks break
cliff_detected <- split_point <= band_values[b]  # split at or below boundary player
```

Verify by comparing within-group variance before and after the split.

**`"gap_ratio"`:**

```r
gaps       <- abs(diff(sort(B_lower_values)))
range_val  <- diff(range(B_lower_values))
gap_ratios <- gaps / range_val
cliff_pos  <- which(gap_ratios >= cliff_gap_ratio_threshold)
cliff_detected <- length(cliff_pos) > 0L
```

**Output** from cliff detection for each position:

| Field | Type | Notes |
|-------|------|-------|
| `cliff_detected` | logical | Whether band was truncated |
| `cliff_location` | integer or NA | Index `j` within `B_lower` where cliff was found |
| `cliff_magnitude` | numeric or NA | Gap size at cliff location |
| `swingman` | logical | Whether any swingman (80 <= IP <= 120) is in the band |
| `n_band_players` | integer | Number of players actually averaged (after truncation) |

### 5.7 Replacement Stat Line Computation

For each position `pos`, after cliff truncation produces final band `B_final`:

**Counting stats** (all non-rate stats):

```r
repl_stat[cat] <- mean(projections[B_final, cat], na.rm = FALSE)
```

**ERA / WHIP** (IP-weighted):

```r
repl_ERA  <- sum(projections[B_final, "ERA"]  * projections[B_final, "IP"]) /
             sum(projections[B_final, "IP"])
repl_WHIP <- sum(projections[B_final, "WHIP"] * projections[B_final, "IP"]) /
             sum(projections[B_final, "IP"])
```

**AVG** (AB-weighted, not mean of AVG values):

```r
repl_AVG <- sum(projections[B_final, "AVG"] * projections[B_final, "AB"]) /
            sum(projections[B_final, "AB"])
```

Equivalently, `sum(H_i) / sum(AB_i)` where `H_i = AVG_i * AB_i`.

**IP column:** Always included for pitcher positions even if IP is not a scored category.

### 5.8 Positional Adjustment Computation (via `compute_positional_adjustments()`)

`compute_positional_adjustments()` in `R/replacement_internal.R` takes:

```r
compute_positional_adjustments(
  replacement_stats,
  config,
  positional_adjustment_method,
  catcher_adjustment_method,
  pos_weight       = NULL,
  sgp_denominators = NULL,
  pass_number      = 1L
)
```

Returns a named numeric vector `scarcity_premium` indexed by position name.

**Steps:**

1. Compute `global_replacement_level` separately for hitters (mean over all `primary_hitter_slots` replacement stats, weighted by `roster_slots[pos]`) and pitchers (mean over SP + RP, weighted by pitcher slot counts).

2. Compute `scarcity_premium[pos] = global_replacement - replacement_stats[pos, ...]` in the method's unit system:
   - `"fvarz"`: compute in z-score units; the replacement player's z-score is subtracted per position
   - `"sgp"`: compute in SGP units; requires `sgp_denominators`; return NULL on pass 1 when denominators not yet available
   - `"dollar"`: compute in dollar-space; each position's contribution above replacement scaled by budget proportion
   - `"posblend"`: `adjusted = pos_weight * (stat - pos_avg) + (1 - pos_weight) * (stat - global_avg)`

3. Apply `catcher_adjustment_method` BEFORE the zero-sum assertion:
   - `"split_pool"` (default): replace `scarcity_premium["C"]` with a catcher-specific value; exclude "C" from `zero_sum_positions`
   - `"positional_default"`: apply `positional_adjustment_method` to catchers unchanged
   - `"partial_offset"`: empirical fraction; for research use
   - `"none"`: `scarcity_premium["C"] <- 0`

4. If `sgp_denominators` supplied but `positional_adjustment_method != "sgp"`: `cli_inform()` when `verbose = TRUE`.

### 5.9 Zero-Sum Assertion

```r
zero_sum_positions <- if (catcher_adjustment_method == "split_pool") {
  setdiff(primary_hitter_slots, "C")
} else {
  primary_hitter_slots
}

# Filter to positions with roster_slots > 0
zero_sum_positions <- intersect(zero_sum_positions,
                                names(config$roster_slots[config$roster_slots > 0]))

check_val <- sum(
  config$roster_slots[zero_sum_positions] * scarcity_premium[zero_sum_positions]
)
if (abs(check_val) > 1e-6) {
  cli::cli_abort(
    c(
      "Zero-sum assertion failed for positional adjustments.",
      "i" = "sum(roster_slots * scarcity_premium) = {check_val}; must be < 1e-6.",
      "i" = "This is an internal computation bug."
    ),
    class = "rotostats_error_zero_sum_violation",
    call  = rlang::caller_env()
  )
}
```

This assertion always runs internally. It is never triggered by user input errors — it represents an internal computation bug.

### 5.10 Boundary Rate Method Ranking

**`"raw_ip"` (default):**

For composite ranking of pitchers (used to sort the pitcher pool before identifying the boundary):

```r
ERA_contribution[i]  <- (repl_ERA  - pitcher_ERA[i])  * pitcher_IP[i]
WHIP_contribution[i] <- (repl_WHIP - pitcher_WHIP[i]) * pitcher_IP[i]
```

For hitters:

```r
AVG_contribution[i] <- (player_AVG[i] - repl_AVG) * player_AB[i]
```

These contributions are added to counting stat z-scores / SGP totals for composite ranking. Rankings are independent of league depth — a pitcher of fixed quality ranks identically in 10-team and 15-team leagues.

`expected_team_IP` and `expected_team_AB` are NOT used in ranking; they belong in `sgp()`.

**`"sgp_pool"`:**

The other N-1 rostered pitchers form the baseline pool; a pitcher's ERA contribution is the marginal change in the pool's combined ERA/WHIP from adding that pitcher. Requires `sgp_denominators`. Partial circularity is managed by the existing iteration loop.

### 5.11 `normalize_to_season`

When `normalize_to_season = TRUE`, normalize counting stats to full-season baselines:

- Hitters: 600 PA (550 AB when PA unavailable)
- SP: 200 IP
- RP: 70 IP

Set `attr(result, "stat_units") = "full_season_normalized"`. When `normalize_to_season = FALSE`, set `attr(result, "stat_units") = "raw_projected"`.

### 5.12 League History Validation

When `league_history` is supplied:

**$1 calibration check:**

- Filter `league_history$prices` to `price <= 1`
- If `is_keeper` present: exclude `is_keeper == TRUE` rows exactly
- If `is_keeper` absent: apply `trim_method` to remove overperformers from the $1 pool:
  - `"iqr"`: remove above Q3 + 1.5 * IQR of end-of-season values among $1 players
  - `"mad"`: remove above median + 3 * MAD
  - `"kde"`: detect trough via KDE; error with `rotostats_error_kde_no_trough` if no trough detectable
- If fewer than `params$calibration_min_n` remain: `cli_warn(..., class = "rotostats_warning_calibration_suppressed")` unconditionally

**Name matching** (when `player_id` absent from `$prices`):

```r
normalize_name <- function(x) {
  x <- stringi::stri_trans_nfd(x)
  x <- stringi::stri_replace_all_regex(x, "\\p{Mn}", "")
  x <- tolower(x)
  x <- gsub("[^a-z0-9 ]", "", x)
  x <- trimws(gsub("\\s+", " ", x))
  x
}
```

Mismatched names: `cli_warn(..., class = "rotostats_warning_name_match_failure")` when `verbose = TRUE`.

**IP/AB divergence validation:**

When `IP` and `AB` present in `league_history$team_season`:

```r
proj_team_IP <- sum(projections$IP, na.rm = TRUE) / config$n_teams
hist_team_IP <- mean(league_history$team_season$IP, na.rm = TRUE)
if (abs(proj_team_IP - hist_team_IP) / hist_team_IP > params$ip_ab_divergence_tol) {
  if (verbose) cli_warn(..., class = "rotostats_warning_team_total_divergence")
}
```

---

## 6. Return Value Contract

`replacement_level()` returns a named list. `format_replacement_output()` (in `R/replacement_internal.R`) constructs and validates this list before return. `assert_replacement_output_contract()` runs as a final internal check.

### 6.1 List Structure

```r
list(
  replacement_stats      = <data.frame>,
  positional_adjustments = <named numeric vector or NULL>,
  cliff_metric           = <data.frame>,
  two_way_players        = <character vector>,
  pool_diagnostics       = <list>,
  method                 = "boundary_band",
  params                 = <list>
)
```

### 6.2 `replacement_stats` Data Frame

One row per position (from `names(config$roster_slots)` that have `roster_slots > 0`, plus SP and RP for pitchers). Columns:

| Column | Type | Notes |
|--------|------|-------|
| `position` | character | Position name, e.g. `"C"`, `"SP"` |
| `[stat]` | numeric | One column per category in `config$categories` |
| `IP` | numeric | Always present for pitcher rows (SP, RP) |
| `AB` | numeric | Present for hitter rows when AVG/OBP/SLG in categories |
| `n_band_players` | integer | Players averaged into this replacement line |
| `cliff_detected` | logical | Whether cliff detection truncated the band |

Stat column names exactly match the uppercase column names in `projections`.

### 6.3 `cliff_metric` Data Frame

One row per position. Columns: `position`, `cliff_detected`, `cliff_location`, `cliff_magnitude`, `swingman`, `n_band_players`.

### 6.4 `params` List

```r
list(
  converged              = <logical>,
  iterations             = <integer>,
  delta                  = <numeric: max(abs(delta_repl_stats)) at final pass>,
  n_teams                = config$n_teams,
  roster_slots           = config$roster_slots,
  band_width             = K,
  cliff_threshold        = cliff_thr,
  sort_by                = sort_by,
  stat_units             = attr(result, "stat_units"),
  catcher_adjustment_method = catcher_adjustment_method,
  method                 = "boundary_band"
)
```

### 6.5 `pool_diagnostics`

```r
list(
  position_sd_ratio = <named numeric vector: pos → ratio of within-pos SD to global SD per scored category>
)
```

### 6.6 Output Attributes

Attach after `format_replacement_output()` constructs the list:

```r
attr(result, "stat_units")           <- stat_units   # "raw_projected" | "full_season_normalized"
attr(result, "config")               <- config
attr(result, "projections")          <- stored_projections   # full projections df, not loop copy
attr(result, "position_assignments") <- new_assignments       # named chr: player_id → position
attr(result, "converged")            <- converged
attr(result, "iterations")           <- pass
attr(result, "delta")                <- delta
```

---

## 7. Internal Helpers (`R/replacement_internal.R`)

### 7.1 Mandated Shared Helpers

**`format_replacement_output(replacement_stats, positional_adjustments, cliff_metric, two_way_players, pool_diagnostics, method, params)`**

Constructs and returns the output list. Called by both `replacement_level()` and `replacement_from_prices()`. Does not attach output attributes (caller does that). Validates list structure before returning.

**`compute_positional_adjustments(replacement_stats, config, positional_adjustment_method, catcher_adjustment_method, pos_weight, sgp_denominators, pass_number)`**

Computes `scarcity_premium` named numeric vector per §5.8. Called by both exported functions. Returns `NULL` when `positional_adjustment_method = "sgp"` and `pass_number == 1L`.

**`assert_replacement_output_contract(result)`**

Validates the complete output object. Checks: required list elements present, `replacement_stats` has required columns, `params` has required keys, `attr(result, "stat_units")` is one of the two valid values. Aborts with informative message on failure. Called as the last step before return in both exported functions.

### 7.2 Additional Internal Helpers (builder discretion on naming)

These helpers are identified by logical function boundary; builder assigns names:

- **Boundary band computer**: given sorted pool, position, K_eff, returns band player indices
- **Cliff detection dispatcher**: calls the appropriate method from §5.6
- **Rate stat replacement line computer**: given band players, returns replacement stat line per §5.7
- **SP/RP role infer + swingman flag**: per §5.2 (must compute swingman BEFORE role)
- **Normalize name**: UTF-8 NFD normalization for `replacement_from_prices()` per §5.12
- **Z-score pool computer**: given projections and pool sizes, computes z-scores for composite ranking
- **Pool sizes delegator**: calls `pool_sizes(config)` from `R/league-config.R` (no duplication of formula)

---

## 8. Algorithm: `replacement_from_prices()`

Fundamentally different algorithm from `replacement_level()`. No projections, no sorting, no band, no cliff detection, no SP/RP inference, no iteration.

**Steps:**

1. Validate inputs per §4.2
2. Filter `prices` to `price <= 1`
3. Exclude `is_keeper == TRUE` rows when column is present; else apply `trim_method`
4. If fewer than `calibration_min_n` remain: `cli_warn("rotostats_warning_calibration_suppressed")`; return `NULL`
5. For each position (inferred from `pos_eligibility` column in `prices` if present, else from `player_type = "batter"/"pitcher"` columns): compute mean end-of-season stat line across remaining $1 players
6. Call `compute_positional_adjustments()` with appropriate arguments
7. Construct output via `format_replacement_output()` with `method = "prices"` in `params`
8. Run `assert_replacement_output_contract(result)`
9. Attach attributes (note: `attr(result, "projections") = NULL` since no projections input)
10. Return

**`prices` required column note:** `replacement_from_prices()` requires a `pos_eligibility` column in `prices` (pipe-delimited, e.g., `"1B|3B"`) to assign players to positions. If this column is absent, the function aborts with `rotostats_error_missing_column`. This requirement must be documented in the roxygen `@param prices` description.

---

## 9. Error Raising Contract

All errors and warnings use `cli::cli_abort()` / `cli::cli_warn()` with `class =` and `call = rlang::caller_env()`. Condition class is the sole routing key; message text is not part of the contract.

| Condition | Class | Level | Verbose-gated? |
|-----------|-------|-------|----------------|
| Missing required column in `projections` | `rotostats_error_missing_column` | error | no |
| Wrong column type | `rotostats_error_wrong_column_type` | error | no |
| Unknown rate stat not in lookup | `rotostats_error_unknown_rate_stat` | error | no |
| `sort_by = "sgp"` + NULL `sgp_denominators` | `rotostats_error_missing_sgp_denominators` | error | no |
| `n_teams * roster_slots[pos]` exceeds pool | `rotostats_error_pool_too_small` | error | no |
| `trim_method = "kde"` no trough | `rotostats_error_kde_no_trough` | error | no |
| `seed_method = "historical_priors"` + NULL `league_history` | `rotostats_error_missing_league_history` | error | no |
| `positional_adjustment_method = "posblend"` + NULL `pos_weight` | `rotostats_error_missing_pos_weight` | error | no |
| Zero-sum violation (internal assertion) | `rotostats_error_zero_sum_violation` | error | no |
| `boundary_rate_method = "sgp_pool"` + `fixed_baseline` denominators | `rotostats_error_rate_method_mismatch` | error | no |
| Convergence not reached at `max_iter` | `rotostats_warning_convergence_not_reached` | warning | **no — always fires** |
| Calibration pool < `calibration_min_n` | `rotostats_warning_calibration_suppressed` | warning | **no — always fires** |
| Team IP/AB diverges > `ip_ab_divergence_tol` | `rotostats_warning_team_total_divergence` | warning | yes (`verbose`) |
| Name match failure in prices | `rotostats_warning_name_match_failure` | warning | yes (`verbose`) |

All 14 new classes must be added to `plans/error-messages.md` by scriber. Builder should not modify that file.

---

## 10. Multi-Position Assignment

### `multi_pos = "highest_par"`

On pass 1: use seed from §5.4 (hierarchy or historical priors).

On pass 2+:

```r
# For each multi-eligible player:
# Compute PAR at each eligible position = stat_line[player] - replacement_stats[pos]
# Assign to position with max PAR
```

Each player appears in exactly one position pool. When a player is reassigned, both the old and new position pool replacement stats update.

### Other multi_pos values

- `"primary"`: Use first position in `pos_eligibility`; ignore `position_assignments`
- `"all"`: Compute per-eligible-position; `replacement_stats` output is player × position × stat
- `"custom"`: Use caller-supplied `position_assignments` named vector on all passes

### Two-Way Players

Compute after convergence:

```r
two_way_PAR[i] = hitter_PAR[i] + pitcher_PAR[i] - 1
```

`two_way_players` = character vector of `player_id` where `hitter_PAR > 0` AND `pitcher_PAR > 0`. Informational only; `dollar_values()` handles slot deduplication.

---

## 11. DESCRIPTION Updates

Builder must update `DESCRIPTION`:

**Add to `Imports`:**
- `checkmate`
- `stringi`

**Add to `Suggests`:**
- `classInt` — used only when `cliff_method = "fisher_jenks"`
- `mixtools` — future `trim_method = "gmm"` (not yet implemented)

`stats` (base R) is used for `stats::mad()` and `stats::quantile()`; no DESCRIPTION entry needed.

---

## 12. Roxygen Documentation Outline

### `replacement_level()`

```r
#' @title Per-position replacement-level stat lines
#' @description <one paragraph from §Why This Measure in spec>
#' @param projections data.frame. One row per projected player. Required columns: ...
#' @param config A `league_config` object from [league_config()].
#' @param sort_by Character. `"zscore"` (default) or `"sgp"`. ...
#' @param sgp_denominators Named numeric vector from [sgp_denominators()]. Required when `sort_by = "sgp"`.
#' @param ... <all other params documented>
#' @return A named list with elements: `replacement_stats`, `positional_adjustments`,
#'   `cliff_metric`, `two_way_players`, `pool_diagnostics`, `method`, `params`.
#'   Carries attributes: `stat_units`, `config`, `projections`, `position_assignments`,
#'   `converged`, `iterations`, `delta`.
#' @seealso [replacement_from_prices()], [default_replacement_params], [rate_stat_denominators()]
#' @examples
#' # minimal reproducible example with two positions and three players
#' @export
```

### `replacement_from_prices()`

```r
#' @title Replacement-level stat lines derived from historical $1 auction prices
#' @description ...
#' @param prices data.frame. Historical auction results. Required columns: `year`,
#'   `player_name`, `price`, `pos_eligibility`. Optional: `player_id`, `is_keeper`.
#' @param n_teams Positive integer.
#' @param roster_slots Named integer vector.
#' @param categories Character vector of scored categories.
#' @param trim_method Character. `"iqr"` (default), `"mad"`, or `"kde"`.
#' @param calibration_min_n Minimum players after trimming before suppression warning.
#' @param verbose Logical.
#' @return Same named list schema as [replacement_level()] with `method = "prices"`.
#'   Returns `NULL` when calibration is suppressed.
#' @seealso [replacement_level()]
#' @export
```

### `default_replacement_params`

```r
#' @title Default parameters for replacement_level()
#' @description Named list of all numeric constants used by [replacement_level()].
#'   Override specific entries: `replacement_params = list(band_width_K = 2L)`.
#' @format A named list with elements: `band_width_K`, `cliff_threshold`,
#'   `cliff_min_n`, `sp_ip_threshold`, `sp_rp_split_default`, `ip_ab_divergence_tol`,
#'   `calibration_min_n`, `convergence_eps`, `convergence_max_iter`.
#' @export
```

### `rate_stat_denominators()`

```r
#' @title Built-in rate-stat denominator lookup
#' @description Returns the named character vector mapping rate-stat category
#'   names to their denominator column (e.g., `ERA -> "IP"`). Supply `rate_denominators`
#'   to [replacement_level()] to add or override entries.
#' @return Named character vector.
#' @export
```

---

## 13. Pre-Commit Checklist

Builder MUST perform these steps before committing any file with roxygen2 changes:

1. `devtools::document()` — regenerates `NAMESPACE` and `man/` files
2. `devtools::check()` — zero errors, zero warnings, zero notes (except standard "no visible binding" for NSE patterns, which must be suppressed via `utils::globalVariables()`)
3. `devtools::test()` — all tests pass

Conventional Commits format: `feat(replacement): <description>`. Branch: `feature/replacement-level` targeting `develop`.

---

## 14. Implementation Notes and Risk Reminders

1. **Iteration loop copies**: Strip `attr(result, "projections")` from any loop-internal objects. Attach projections only to the final returned list.

2. **`pool_sizes()` duplication**: Call `pool_sizes(config)` from `R/league-config.R` directly. Do NOT copy the formula into `R/replacement_internal.R` or `R/replacement.R`.

3. **Swingman before role**: Compute `swingman_flag` vector using raw IP before assigning roles. A pitcher later reclassified by pool membership still carries the swingman flag.

4. **`sgp()` forwarding**: When calling `sgp()` internally (`sort_by = "sgp"`, pass 2+), forward `league_history` (if supplied by user) and `config` to satisfy `sgp()`'s `blended_pool` contract.

5. **Zero-sum assertion fires AFTER `catcher_adjustment_method` override**: The `catcher_adjustment_method` replacement of `scarcity_premium["C"]` must complete before the zero-sum check runs. The two steps are sequential.

6. **`positional_adjustments` NULL on pass 1 with `"sgp"` method**: `compute_positional_adjustments()` returns `NULL` when `positional_adjustment_method = "sgp"` and `pass_number = 1L`. The output list must accept `NULL` for this field. `assert_replacement_output_contract()` must allow `NULL` only in this case.

7. **Vectorized — no row-wise loops**: All per-player computations must be vectorized. No `for (i in seq_len(nrow(projections)))` loops over players.

8. **`classInt` conditional**: `cliff_method = "fisher_jenks"` requires `classInt`. Use `requireNamespace("classInt", quietly = TRUE)` and abort with `cli_abort()` if not installed.
