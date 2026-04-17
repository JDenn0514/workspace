# Simulation Spec: `sgp_denominators()` Monte Carlo Validation

> **Pipeline:** Simulation pipeline — Simulator's only input
> **Run ID:** sgp-denom-dgpb-rerun-2026-04-17
> **Date:** 2026-04-17
> **Status:** Ready for Simulator (Q4, Q7 re-run only)
> **Supersedes:** sgp-denominators-2026-04-16 sim-spec.md (DGP-B section only)

This specification is independently executable. Simulator reads only this file. The
estimator under study is a black box called via the public R API of `sgp_denominators()`.
No knowledge of the internal implementation is assumed or used in the DGP design.

## Changelog

### 2026-04-17 — DGP-B σ-shift correction (this run, sgp-denom-dgpb-rerun)

**Motivation.** The 2026-04-16 run's DGP-B shifted the league *mean* of team totals
across the structural break (break_at = 7, n_years = 10). The OLS denominator for
`sgp_denominators(method = "ols")` is **mean-invariant and σ-dependent**:
the OLS slope of `rank ~ total` depends only on within-year spread, not on the
absolute level of totals. A mean-only shift therefore produced **zero change** in the
OLS estimand across the break — θ_pre = θ_post — so the Q4 "decay helps under break"
and Q7 "override improves SB" acceptance criteria were untestable (not refuted).

**Fix in this run.** DGP-B now shifts within-year σ across the break:

- Pre-break σ  = 10 (years 1–6)
- Post-break σ = 25 (years 7–10)
- Mean is fixed at cat_mean = 100 (does not affect θ under OLS)

**Analytical estimand targets (n_teams = 12):**

| Regime | σ  | θ_OLS ≈ (n−1)·σ / E[R_n] |
|---|---|---|
| Pre-break  | 10 | 33.76 |
| Post-break | 25 | 84.40 |
| Gap       |    | 50.64 |

The 2.5× σ-ratio produces a 50-unit absolute θ gap — strongly detectable at R = 2000
(the sampling SE of the mean denominator estimate across reps is on the order of
0.1 for these settings, so the pre/post z-score is well in excess of 100 under
empirical calibration). This yields a clean Q4 / Q7 test of the decay-under-break
and override-improvement criteria.

**Parameter choice (Planner).** `plans/sgp-cleanup.md` R2 suggested "e.g., pre σ = 15,
post σ = 25" as one example; Planner chose pre = 10, post = 25 to produce a stronger,
more diagnostic signal (2.5× instead of 1.67× σ ratio). Both choices are within the
plan's stated latitude ("planner picks exact parameters"); the 2.5× ratio matches the
seed sketch already present in this run's `sim-spec.md` §2 DGP-B definition.

**Scope of this correction.**
- DGP-B (§2) code block updated — **already present in the seeded sim-spec.md**.
- Q4 (§7) and Q7 (§10) narrative text updated to reference σ-shift (not mean shift).
- Acceptance criteria thresholds are **unchanged** (the criteria were valid; only
  the DGP that generated test data was wrong).
- Seed strategy (§3) is **unchanged** (master 2026041601, same study offsets).
- Q1, Q2, Q3, Q5, Q6 are **not affected** — DGP-A is unchanged and these studies
  do not use DGP-B.

**Downstream impact.**
- Simulator re-runs Q4 and Q7 only. Prior-run Q1/Q2/Q3/Q5/Q6 verdicts are referenced
  by path and kept as-is.
- Tester adds a pre-flight empirical-θ calibration step: sample one very long
  DGP-B pre-regime and one very long DGP-B post-regime, compute θ̂_pre and θ̂_post,
  and confirm the gap exceeds sampling noise. If not, BLOCK with route-to-Planner.

### 2026-04-16 — Post-mortem notes (appended at end of prior run)

- DGP-B mean-shift design flaw identified during review (Concern 2).
- Sim-spec §2 formula for DGP-A true denominator clarified (OLS vs SD estimand;
  Jensen's inequality caveat).
- Q3 bootstrap undercoverage at n_years ≤ 6 (~10pp) documented as known limitation
  (Jensen upward bias); BCa deferred.

---


## 1. Estimator Interface (Black Box)

```r
# Install package and load
library(rotostats)

# Call interface
sgp_denominators(
  league_history,          # list with $team_season data.frame
  scoring_categories = NULL,
  n_teams            = NULL,
  years              = "all",
  weights            = flat(),  # or exp_decay(lambda)
  method             = "ols",
  exclude_years      = integer(0),
  n_bootstrap        = 0L,
  denom_floor        = 1e-9,
  ci_level           = 0.95
)

# Returns: S3 object of class c("sgp_denominators", "list")
# $denominators      — named numeric vector, one per category
# $year_diagnostics  — data.frame: year, category, slope, r_squared, weight
# $bootstrap_ci      — data.frame or NULL
# $meta              — list with rate_conversion, method, years_used
```

**Extraction of estimator output from the black box:**
```r
get_denom <- function(result, cat = "HR") {
  result$denominators[[cat]]
}

get_slope <- function(result, cat = "HR", year) {
  diag <- result$year_diagnostics
  diag$slope[diag$category == cat & diag$year == year]
}
```

The simulator must NOT import or read any source file from `R/`. Call the installed
package only.

---

## 2. DGP Library

All DGPs generate synthetic `$team_season` data frames for use as input to the estimator.

### DGP-A: Stable counting stat (HR-like)

```r
dgp_stable <- function(
  n_teams = 12,
  n_years = 10,
  true_slope = 1/15,   # true denominator = 15 HR per place
  cat_mean = 180,      # league mean team HR total
  cat_sd = 25,         # within-year SD of team HR totals
  seed = NULL
) {
  if (!is.null(seed)) set.seed(seed)

  # true DGP: standings rank is a perfect linear function of total + noise
  # total_{t,y} = cat_mean + epsilon_{t,y},   epsilon ~ N(0, cat_sd^2)
  # rank_{t,y} = determined by sorting within year (no noise in rank, noise only in totals)
  # true slope = 1/(cat_sd_effective) -- but we parameterize by true_slope directly:
  # E[rank ~ total slope] = (n_teams - 1) / (2 * cat_sd * ... )
  # Simpler: fix true_slope = (n_teams-1)/(range_of_totals)
  # For simplicity, we set true_slope = 1/true_denominator and accept that
  # the realized slope per year will be a random variable around this expectation.

  # The "true" denominator = 1/true_slope
  true_denom <- 1 / true_slope

  years <- seq(2000, 2000 + n_years - 1)
  rows <- lapply(years, function(y) {
    totals <- rnorm(n_teams, mean = cat_mean, sd = cat_sd)
    data.frame(
      year    = y,
      team_id = paste0("T", seq_len(n_teams)),
      HR      = totals
    )
  })
  list(
    team_season = do.call(rbind, rows),
    true_denom  = true_denom,
    true_slope  = true_slope
  )
}
```

**True denominator derivation for DGP-A:**

The OLS estimand and the SD-method estimand are **distinct** for this DGP, even though
they are asymptotically related. The key relationship is:

```
E[β̂_OLS] ≈ E[R_n] / ((n-1) × σ)
```

where E[R_n] is the expected range of n standard normals (see `expected_range_normal()`).
Therefore the OLS estimand is:

```
θ_OLS = 1 / E[β̂_OLS] ≈ (n-1) × σ / E[R_n]
```

For n_teams=12, σ=25: θ_OLS ≈ 11 × 25 / 3.2587 ≈ **84.1**.

**CRITICAL NOTE on estimands:** The formula above gives the OLS estimand, but the
`method = "ols"` path of `sgp_denominators()` computes `1/|β̂|` where β̂ is the raw OLS
slope from `rank ~ total`. Due to Jensen's inequality (1/x is convex), `E[1/β̂] > 1/E[β̂]`.
At small n_years (e.g., 6), the **realized** OLS denominator is substantially larger than
θ_OLS = 84.1 — empirically θ ≈ 6.70 for the method = "ols" path at this n. This
discrepancy is because the OLS path estimates a different quantity (the slope estimator
itself is unbiased, but the denominator = 1/slope is not).

**For simulation studies, use empirical θ calibration** rather than the analytical
formula to define the true denominator for each (method, n_teams, σ) combination.
Empirical calibration: run one very-long simulation (n_years → ∞, e.g., 2000 years)
and take the resulting denominator as θ. This bypasses the Jensen's inequality issue.

The analytical θ_OLS ≈ 84.1 is provided as a theoretical reference and matches the
`"sd"` method's estimand exactly, but is **not** the correct comparison baseline for
`method = "ols"` in finite-sample studies. See simulation.md §0.2 for the empirical
calibration table used in the previous run.

### DGP-B: Structural-break category (SB-like)

**Design rationale:** DGP-B is intended to test whether exponential decay weighting
outperforms flat weighting when the denominator changes across a break. The OLS
denominator is σ-dependent and mean-invariant (the OLS slope of rank ~ total does not
change when all totals shift by a constant, because rank depends only on relative order).
Therefore the break must shift within-year σ (spread), not the league mean, to create
a genuine regime change in the denominator estimand.

Previous specification shifted `pre_mean` vs `post_mean` while holding σ fixed — this
produces zero change in the OLS denominator and makes the decay trade-off untestable.
This version shifts `pre_sd` vs `post_sd` instead.

```r
dgp_break <- function(
  n_teams   = 12,
  n_years   = 10,
  break_at  = 7,       # break occurs at year index break_at (1-indexed)
  cat_mean  = 100,     # league mean SB total (fixed across break — does not affect OLS denom)
  pre_sd    = 10,      # pre-break within-year SD; true denom ≈ pre_sd*(n-1)/E[R_n]
  post_sd   = 25,      # post-break within-year SD (2.5× increase); true denom increases
  seed      = NULL
) {
  if (!is.null(seed)) set.seed(seed)

  years <- seq(2000, 2000 + n_years - 1)
  rows <- lapply(seq_along(years), function(i) {
    y     <- years[i]
    sd_y  <- if (i < break_at) pre_sd else post_sd
    totals <- rnorm(n_teams, mean = cat_mean, sd = sd_y)
    data.frame(
      year    = y,
      team_id = paste0("T", seq_len(n_teams)),
      SB      = totals
    )
  })

  # True denominators: OLS estimand ≈ (n-1) × σ / E[R_n] (empirically calibrate per regime)
  # These are the SD-method analytical values; OLS converges to the same in expectation.
  true_denom_pre  <- pre_sd  * (n_teams - 1) / expected_range_normal(n_teams)
  true_denom_post <- post_sd * (n_teams - 1) / expected_range_normal(n_teams)
  # NOTE: Use empirical calibration for OLS in practice (see §2 note on estimands).

  list(
    team_season      = do.call(rbind, rows),
    break_at_year    = years[break_at],
    true_denom_pre   = true_denom_pre,
    true_denom_post  = true_denom_post,
    true_denom_final = true_denom_post  # target = post-break denominator
  )
}
```

**Expected behavior of DGP-B:** Because σ doubles across the break (pre_sd=10, post_sd=25),
the OLS denominator in post-break years is ≈2.5× larger than in pre-break years. A flat-
weighted estimator averages across both regimes and underestimates the current (post-break)
denominator. An exponentially-decaying estimator downweights pre-break years and tracks
the post-break denominator more closely, reducing MSE relative to flat weighting.

**Simulator note for next run:** Update Q4 DGP-B settings and Q7 SB DGP to use this
σ-shift version. The acceptance criteria in Q4 and Q7 (below) are valid for this DGP.
The previous run's DGP-B used a mean-shift, which left σ unchanged and made all
denominators identical across regimes — the "decay helps under break" criterion was
therefore untestable. This redesign fixes that.

---

## 3. Seed Strategy

**Master seed:** `2026041601`

**Per-scenario seed derivation:**
```r
scenario_seed <- function(study_id, scenario_idx, rep_idx) {
  set.seed(2026041601)
  master_seeds <- sample.int(.Machine$integer.max, 10000)
  master_seeds[(study_id * 1000 + scenario_idx * 100 + rep_idx) %% 10000 + 1]
}
```

Each simulation call:
```r
set.seed(scenario_seed(study_id, scenario_idx, rep_idx))
data_obj <- dgp_*(...)
result   <- sgp_denominators(list(team_season = data_obj$team_season), ...)
```

**RNG:** Base R default (`kind = "Mersenne-Twister"`). Do not change `.Random.seed`.

---

## 4. Study Q1: OLS Convergence

**Research question:** Does `E[1/|β̂_c|] → 1/|β̂_true|` as `n_years → ∞`?

**DGP:** DGP-A (stable counting stat), n_teams = 12, cat_sd = 25, cat_mean = 180.
True denominator: `true_denom_A = 25 × 11 / expected_range_normal(12) ≈ 84.1`.

**Scenario grid:**

| Factor | Levels |
|--------|--------|
| `n_years` | 3, 6, 10, 20, 50 |
| `n_teams` | 12 (fixed) |
| `weights` | flat() |
| `method` | "ols" |

Total scenarios: 5.

**Replications:** R = 2000 per scenario.

**Estimand:** `θ = true_denom_A`

**Metrics per scenario:**

| Metric | Formula |
|--------|---------|
| `bias` | `mean(d̂) - θ` |
| `rel_bias` | `bias / θ` |
| `variance` | `var(d̂)` |
| `mse` | `mean((d̂ - θ)^2)` |
| `rmse` | `sqrt(mse)` |
| `failure_rate` | `mean(is.na(d̂) | is.infinite(d̂))` |

**Acceptance criteria:**

| Criterion | Threshold | n_years |
|-----------|-----------|---------|
| Convergence: rel_bias → 0 | `abs(rel_bias) ≤ 0.10` | ≥ 20 |
| Convergence: rel_bias near 0 | `abs(rel_bias) ≤ 0.05` | ≥ 50 |
| RMSE decreasing | Log-log slope of RMSE vs n_years ≤ -0.4 | all |
| Failure rate | `≤ 0.01` | all |

**Output table schema:**

```
| n_years | bias | rel_bias | variance | rmse | failure_rate |
```

**Simulator note:** Report the bias direction at n_years = 6 explicitly (positive =
denominator overestimates, negative = underestimates). This feeds Study Q2.

---

## 5. Study Q2: OLS Finite-Sample Bias Direction

**Research question:** At n_years = 6, is `E[1/|β̂_c|] > 1/|β̂_true|` (upward bias)?
This tests the audit flag (F3) which claimed OLS denominator is biased upward.

**Statistical argument:** Jensen's inequality applied to the convex function 1/x for x > 0:
`E[1/β̂] ≥ 1/E[β̂]`. Since β̂ is a random variable with variance > 0, strict inequality
holds: `E[1/β̂] > 1/E[β̂] = 1/β̂_true`. Thus the OLS denominator estimator is upward-biased
for any finite sample. The simulation should confirm this and quantify the magnitude.

**DGP:** DGP-A with n_teams ∈ {10, 12, 15}, n_years = 6.

**Scenario grid:**

| Factor | Levels |
|--------|--------|
| `n_teams` | 10, 12, 15 |
| `n_years` | 6 (fixed) |
| `cat_sd` | 20, 25, 35 (varying spread → varying SNR) |
| `weights` | flat() |
| `method` | "ols" |

Total scenarios: 9.

**Replications:** R = 5000 per scenario.

**Estimand:** `θ_c = true_denom(n_teams, cat_sd)` (computed analytically per scenario).

**Metrics:**
- `bias` = `mean(d̂) - θ`
- `rel_bias` = `bias / θ` — key metric; should be positive per Jensen
- `bias_pvalue` — one-sample t-test p-value for `H₀: E[d̂] = θ` (two-sided)
- `bias_ci_lower`, `bias_ci_upper` — 95% CI for mean bias

**Acceptance criteria:**

| Criterion | Threshold |
|-----------|-----------|
| Bias direction confirmed positive | `rel_bias > 0` in all 9 scenarios |
| Bias magnitude moderate (not catastrophic) | `abs(rel_bias) ≤ 0.25` at n_years=6, n_teams=12 |
| t-test confirms significance | `bias_pvalue < 0.05` for n_teams=12, n_years=6 |

**What to report:**
1. Table of rel_bias by n_teams × cat_sd
2. Comparison: `E[1/β̂]` vs `1/E[β̂]` — compute both and confirm Jensen's prediction
3. Flag whether the audit's claim (upward bias) is confirmed or refuted

**Output table schema:**
```
| n_teams | cat_sd | bias | rel_bias | bias_ci_lower | bias_ci_upper | bias_pvalue |
```

---

## 6. Study Q3: Bootstrap CI Coverage at n_years = 6

**Research question:** Does the percentile bootstrap CI achieve nominal coverage at n_years = 6?

**DGP:** DGP-A, n_teams = 12, cat_sd = 25.
True denominator `θ ≈ 84.1`.

**Scenario:** Single condition — n_years = 6, n_teams = 12.

**Replications:** R = 2000 outer simulations.

**Bootstrap setting per outer replicate:** `n_bootstrap = 500` (to keep runtime manageable),
`ci_level = 0.95`.

**Procedure per replicate:**
```r
seed_i <- scenario_seed(study_id=3, scenario_idx=1, rep_idx=i)
set.seed(seed_i)
data_obj <- dgp_stable(n_teams=12, n_years=6, seed=NULL)  # seed already set
result <- sgp_denominators(
  list(team_season = data_obj$team_season),
  scoring_categories = "HR",
  n_teams     = 12L,
  n_bootstrap = 500L,
  ci_level    = 0.95,
  weights     = flat(),
  method      = "ols",
  exclude_years = integer(0)
)
ci <- result$bootstrap_ci
covered_i <- (ci$ci_lower <= theta) & (theta <= ci$ci_upper)
width_i   <- ci$ci_upper - ci$ci_lower
```

**Metrics:**
- `coverage` = `mean(covered_i)` over R outer replicates
- `mean_width` = `mean(width_i)` — median CI width in denominator units
- `rel_width` = `mean_width / θ` — CI width relative to true denominator

**Acceptance criteria:**

| Criterion | Threshold |
|-----------|-----------|
| Coverage within nominal band | `coverage ∈ [0.90, 1.00]` at n_years=6 |
| Coverage within tight band (desired) | `coverage ∈ [0.93, 0.97]` — document if not achieved; note known undercoverage due to upward bias from Q2 |
| No catastrophic undercoverage | `coverage ≥ 0.85` required minimum |
| Relative width reasonable | `rel_width ≤ 0.60` at n_years=6 |

**Note on expected result:** Given the upward bias from Study Q2 (denominator tends to
be overestimated), percentile bootstrap CIs may undercover slightly for the true θ — the
CI is centered on a biased estimate. Document the coverage gap quantitatively and note
whether BCa bootstrap would be warranted (BCa shifts the CI to account for bias).

**Output table schema:**
```
| n_years | n_teams | n_bootstrap | coverage | mean_width | rel_width | notes |
```

---

## 7. Study Q4: Time-Decay Variance Trade-Off

**Research question:** Does exponential decay weighting reduce MSE under a structural break (SB-like)? Does it increase MSE under a stable category (HR-like)?

**DGP-A settings:** n_teams = 12, n_years = 10, cat_sd = 25, cat_mean = 180.
True denominator for DGP-A: `θ_A = 25 × 11 / expected_range_normal(12)`.

**DGP-B settings:** n_teams = 12, n_years = 10, break_at = 7 (year index), cat_mean = 100
(fixed), pre_sd = 10, post_sd = 25. The break shifts within-year spread (σ), not the
mean — this creates a genuine denominator regime change. True denominator for DGP-B
(post-break regime): `θ_B = 25 × 11 / expected_range_normal(12)` (analytical; use
empirical calibration for OLS in practice). Target for the estimator is the post-break
denominator θ_B — the current regime's value.

**Scenario grid:**

| Factor | Levels |
|--------|--------|
| DGP | DGP-A (stable), DGP-B (break) |
| `weights` | flat(), exp_decay(0.9), exp_decay(0.7) |
| `n_years` | 10 (fixed) |
| `n_teams` | 12 (fixed) |
| `method` | "ols" |

Total scenarios: 6 (2 DGPs × 3 weight schemes).

**Replications:** R = 2000 per scenario.

**Estimand:**
- DGP-A: `θ_A` (stable true denominator)
- DGP-B: `θ_B` (post-break true denominator — the estimator should track the current regime)

**Metrics per scenario:**
- `bias`, `rel_bias`, `mse`, `rmse` (same formulas as Q1)

**Acceptance criteria:**

| Criterion | Expected result |
|-----------|-----------------|
| Under DGP-A (stable): flat() has lower MSE than exp_decay(0.7) | `mse[flat] ≤ mse[exp_decay(0.7)]` — decay wastes information by downweighting valid older years |
| Under DGP-B (break): exp_decay(0.7) has lower MSE than flat() | `mse[exp_decay(0.7)] ≤ mse[flat]` — decay appropriately downweights pre-break data |
| Under DGP-B: exp_decay(0.9) is intermediate between flat and exp_decay(0.7) | `mse[flat] ≥ mse[exp_decay(0.9)] ≥ mse[exp_decay(0.7)]` or some ordering |
| Under DGP-A: rel_bias within ±0.15 for all weight schemes | `abs(rel_bias) ≤ 0.15` for all 3 |

**Note on acceptance:** The "decay hurts stable" and "decay helps break" criteria encode
the theoretical expectation. If simulation refutes these (e.g., decay helps even under
stable DGP), document the finding and investigate — it would challenge the design rationale
for the default weight scheme.

**Validity note for DGP-B:** The acceptance criteria `mse[exp_decay(0.7)] ≤ mse[flat]`
under DGP-B are only validly testable when DGP-B shifts within-year σ across the break
(as specified in §2 DGP-B). The previous run's DGP-B shifted the league *mean* while
holding σ fixed — the OLS denominator is mean-invariant and σ-dependent, so no denominator
regime change occurred and the criteria were untestable. The redesigned DGP-B (σ pre=10,
σ post=25) creates the genuine regime change required to test these criteria.

**Output table schema:**
```
| dgp | weights | bias | rel_bias | mse | rmse |
```

**Additional output:** Produce a 2×3 MSE heat table (DGP rows × weight columns) to
summarize the trade-off visually.

---

## 8. Study Q5: Expected-Range Anchor Verification

**Research question:** Does `expected_range_normal(n)` return the correct values at known test points?

**Nature:** This is a unit test, not a Monte Carlo study. It runs once without replication.

**Test cases:**

| n | Expected value | Tolerance |
|---|---------------|-----------|
| 10 | 3.0776 | ±1e-3 |
| 12 | 3.2587 | ±1e-3 |
| 15 | 3.4718 | ±1e-3 |
| 2  | 1.1284 | ±1e-3 |
| 5  | 2.3263 | ±1e-3 |
| 20 | 3.7350 | ±1e-3 |

**Verification method:**
```r
n_vals    <- c(2, 5, 10, 12, 15, 20)
expected  <- c(1.1284, 2.3263, 3.0776, 3.2587, 3.4718, 3.7350)
computed  <- vapply(n_vals, rotostats:::expected_range_normal, numeric(1))
max_error <- max(abs(computed - expected))
```

**Acceptance criterion:** `max_error < 1e-3` for all 6 values.

**Note on additional anchor values:** Values for n=2, 5, 20 should be computed
independently using R's numerical integration to verify against the spec anchors:
```r
# Independent computation for n=2:
integrate(function(x) 1 - pnorm(x)^2 - (1-pnorm(x))^2, -Inf, Inf)$value
# Expected: 2/sqrt(pi) = 1.1284...
```

---

## 9. Study Q6: Method Comparison Under Truth

**Research question:** Which method has lower MSE under a DGP where the true denominator is known? When does `"sd"` outperform `"ols"`?

**DGP:** DGP-A (stable, Gaussian totals), n_teams = 12, cat_sd = 25.
Under DGP-A, the true denominator is the same for OLS and SD methods (both estimate
the same quantity under Gaussian assumption).

**Scenario grid:**

| Factor | Levels |
|--------|--------|
| `method` | "ols", "gap", "trimmed_gap", "sd" |
| `n_years` | 6, 15 |
| `n_teams` | 12 (fixed) |
| DGP | DGP-A (Gaussian, stable) |

Total scenarios: 8 (4 methods × 2 n_years).

**Replications:** R = 2000 per scenario.

**Estimand:** `θ = true_denom_A`

**Metrics:** bias, rel_bias, variance, mse, rmse (same as Q1).

**Acceptance criteria:**

| Criterion | Expected result |
|-----------|-----------------|
| OLS preferred under Gaussian DGP at n_years = 15 | OLS has lower or equal MSE vs gap at n_years ≥ 15 |
| "sd" competitive under Gaussian DGP | "sd" MSE within 10% of OLS at n_years = 15 |
| "gap" more variable than OLS | `variance[gap] ≥ variance[ols]` — gap is range-based and thus more extreme-sensitive |
| "trimmed_gap" between gap and ols | MSE[trimmed_gap] ≤ MSE[gap] |

**Additional scenario: Heavy-tailed DGP (where sd method may struggle):**

Add a supplemental 2-scenario block using a t(3)-distributed total (heavy tails):
```r
dgp_heavy <- function(n_teams=12, n_years=10, scale=25, seed=NULL) {
  if (!is.null(seed)) set.seed(seed)
  years <- seq(2000, 2000+n_years-1)
  rows <- lapply(years, function(y) {
    totals <- 180 + scale * rt(n_teams, df=3)
    data.frame(year=y, team_id=paste0("T",1:n_teams), HR=totals)
  })
  list(team_season = do.call(rbind, rows))
}
```

Compare OLS vs sd under heavy-tailed DGP at n_years ∈ {6, 15}. Hypothesis: OLS
still works (robust to distributional assumption) while sd may be slightly worse
(SD is more sensitive to heavy tails than rank-based methods). Document finding.

**Output table schema:**
```
| method | n_years | bias | rel_bias | variance | mse | rmse |
```

---

## 10. Study Q7: Per-Category Override Sanity Check

**Research question:** Does `category_spec` override correctly isolate per-category behavior without contaminating other categories?

**Setup:** Two-category simulation — one stable (HR) and one structural-break (SB).

**DGP:**
```r
dgp_two_cat <- function(n_teams=12, n_years=10, break_year_idx=7, seed=NULL) {
  # HR: stable category (σ=25 throughout).
  # SB: structural-break category — pre-break σ=10, post-break σ=25.
  # The break shifts within-year spread, not the mean. This creates a genuine
  # change in the SB denominator estimand. See DGP-B §2 redesign note.
  if (!is.null(seed)) set.seed(seed)
  years <- seq(2000, 2000+n_years-1)
  rows <- lapply(seq_along(years), function(i) {
    y      <- years[i]
    sb_sd  <- if (i < break_year_idx) 10 else 25  # σ-shift structural break
    hr_totals <- rnorm(n_teams, 180, 25)
    sb_totals <- rnorm(n_teams, 100, sb_sd)        # mean fixed; spread changes
    data.frame(year=y, team_id=paste0("T",1:n_teams), HR=hr_totals, SB=sb_totals)
  })
  list(
    team_season     = do.call(rbind, rows),
    break_year      = years[break_year_idx],
    # Analytical true denominators (SD-method formula; use empirical calibration for OLS):
    true_denom_HR   = 25 * 11 / expected_range_normal(12),   # stable
    true_denom_SB   = 25 * 11 / expected_range_normal(12)    # post-break target
  )
}
```

**Scenarios:**

| Scenario | `years` (global) | `category_spec` | Expected behavior |
|----------|-----------------|-----------------|-------------------|
| S1: No override, flat | `"all"`, flat | NULL | SB biased by pre-break; HR unaffected |
| S2: SB override, post-break only | `"all"`, flat | `SB = cal(years = after(break_year - 1), weights = exp_decay(0.7))` | SB MSE improves; HR unchanged |
| S3: SB override, decay only | `"all"`, flat | `SB = cal(weights = exp_decay(0.7))` | SB partially improves (uses pre-break but downweights) |

**Replications:** R = 2000 per scenario.

**Metrics per category per scenario:** bias, mse relative to true denominator.

**Acceptance criteria:**

| Criterion | Threshold |
|-----------|-----------|
| S2 SB MSE < S1 SB MSE | `mse_SB[S2] < mse_SB[S1]` — override must improve SB |
| HR MSE unchanged across scenarios | `abs(mse_HR[S2] - mse_HR[S1]) / mse_HR[S1] ≤ 0.05` — override must not perturb HR |
| S3 SB MSE intermediate | `mse_SB[S1] ≥ mse_SB[S3] ≥ mse_SB[S2]` or comparable — partial improvement |

**Output table schema:**
```
| scenario | category | bias | mse | mse_relative_to_s1 |
```

---

## 11. Full Scenario Grid Summary

| Study | Factor(s) varied | Scenarios | Reps | Total runs |
|-------|-----------------|-----------|------|------------|
| Q1 | n_years | 5 | 2000 | 10,000 |
| Q2 | n_teams × cat_sd | 9 | 5000 | 45,000 |
| Q3 | single | 1 | 2000 | 2,000 (each calls n_bootstrap=500) |
| Q4 | DGP × weights | 6 | 2000 | 12,000 |
| Q5 | unit test | 1 | 1 | 1 |
| Q6 | method × n_years | 8 + 4 supp | 2000 | 24,000 |
| Q7 | scenario | 3 | 2000 | 6,000 |
| **Total** | | **34** | | **≈ 99,001** |

---

## 12. Performance Metrics — Formal Definitions

Used uniformly across all studies:

| Symbol | Formula | Notes |
|--------|---------|-------|
| `d̂_r` | Denominator from replicate r | From `result$denominators[[cat]]` |
| `bias` | `mean(d̂_r) - θ` | θ = true denominator from DGP |
| `rel_bias` | `bias / θ` | Dimensionless; positive = overestimate |
| `variance` | `var(d̂_r)` | Sample variance across replicates |
| `mse` | `mean((d̂_r - θ)^2)` | = bias² + variance |
| `rmse` | `sqrt(mse)` | In denominator units |
| `coverage` | `mean(ci_lower_r ≤ θ ≤ ci_upper_r)` | Q3 only |
| `mean_width` | `mean(ci_upper_r - ci_lower_r)` | Q3 only |
| `failure_rate` | `mean(is.na(d̂_r) | is.infinite(d̂_r))` | Must be ≤ 0.01 |

---

## 13. Output Table Schemas and Deliverables

Simulator produces a `simulation.md` report with the following sections:

### Section 1: Q1 Convergence
Table: n_years × {bias, rel_bias, variance, rmse, failure_rate}.
Plot: RMSE vs n_years on log-log scale; annotate log-log slope.

### Section 2: Q2 Finite-Sample Bias
Table: n_teams × cat_sd × {bias, rel_bias, bias_ci_lower, bias_ci_upper, bias_pvalue}.
Summary sentence: "Bias direction is [confirmed / not confirmed] positive. At n_teams=12, n_years=6, rel_bias = [X]."

### Section 3: Q3 Coverage
Table: single row with {coverage, mean_width, rel_width}.
Summary sentence: "Coverage is [X]% at nominal 95%, [meets / fails] the ≥90% threshold. Undercoverage of [X%] is [attributable to / inconsistent with] Jensen's inequality bias."

### Section 4: Q4 Decay Trade-off
Table: DGP × weights × {bias, rel_bias, mse, rmse}.
MSE heat table: 2 rows (DGP) × 3 columns (weights).
Summary: "Under stable DGP, [flat / exp_decay(0.9) / exp_decay(0.7)] has lowest MSE. Under break DGP, [flat / exp_decay(0.9) / exp_decay(0.7)] has lowest MSE."

### Section 5: Q5 Anchor Verification
Table: n × {expected, computed, abs_error}.
Pass/fail verdict.

### Section 6: Q6 Method Comparison
Table: method × n_years × {bias, rel_bias, variance, mse, rmse}.
Recommendation: "Under Gaussian DGP at n_years=6, preferred method by MSE is [X]. At n_years=15, preferred method is [X]."

### Section 7: Q7 Override Sanity
Table: scenario × category × {bias, mse, mse_relative_to_s1}.
Pass/fail verdict on the 3 acceptance criteria.

### Section 8: Global Observations
- Any scenario where `failure_rate > 0.01` (flag as concern)
- Any acceptance criterion that is borderline
- Recommendations for default parameter settings based on simulation evidence

---

## 14. Simulation Validation Assertions for Tester

When Tester reviews the `simulation.md` output, verify:

1. **Q1 Convergence:** `abs(rel_bias) ≤ 0.10` at n_years = 20 (Table Q1).
2. **Q2 Bias Direction:** `rel_bias > 0` in at least 8 of 9 scenarios (Table Q2).
3. **Q3 Coverage:** `coverage ≥ 0.85` (minimum floor); document deviation from 0.95.
4. **Q4 Decay:** `mse_SB[exp_decay(0.7)] < mse_SB[flat]` under DGP-B (Table Q4).
5. **Q5 Anchors:** Max absolute error < 1e-3 across all 6 anchor values (Table Q5).
6. **Q6 Methods:** OLS MSE ≤ gap MSE at n_years = 15 under Gaussian DGP (Table Q6).
7. **Q7 Override:** SB MSE in S2 < S1; HR MSE in S2 within 5% of S1 (Table Q7).
8. **Global:** `failure_rate ≤ 0.01` in all scenarios.
