# Simulation Spec: Monte Carlo Validation of `replacement_level()`

Request ID: replacement-2026-04-16
Agent: planner → simulator
Date: 2026-04-17
Profile: r-package

This spec is the sole input to the simulator pipeline. Simulator does not see `spec.md` or `test-spec.md`. All DGP details and acceptance criteria are self-contained here.

---

## 0. Purpose

Validate the `replacement_level()` function via Monte Carlo simulation across five orthogonal study designs:

1. **Study A** — Boundary-band stability: K=3 vs K=1 year-over-year variance
2. **Study B** — Zero-sum invariant under all four `catcher_adjustment_method` values
3. **Study C** — Convergence of the multi-position reassignment loop
4. **Study D** — Dynamic K cap behavior in thin AL-only pools (12-team AL SS and C)
5. **Study E** — Rank invariance under `boundary_rate_method = "raw_ip"` across league sizes

Each study has a dedicated DGP section (§2), scenario grid (§4), performance metrics (§5), and acceptance criteria (§6).

---

## 1. Estimator Interface

Simulator calls `replacement_level()` as a black box. Do not inspect or alter the implementation.

### Function call template

```r
result <- replacement_level(
  projections = <synthetic_projections_df>,
  config      = <league_config_object>,
  sort_by     = <"zscore" | "sgp">,
  band_width  = <K>,
  catcher_adjustment_method = <method>,
  multi_pos   = <"highest_par" | "primary">,
  max_iter    = 25L,
  tol         = 0.01,
  verbose     = FALSE
)
```

### Key return values used by simulator

| Access path | Type | Description |
|-------------|------|-------------|
| `result$replacement_stats` | data.frame | One row per position; columns include stat names, `n_band_players`, `cliff_detected` |
| `result$positional_adjustments` | named numeric | `scarcity_premium[pos]` indexed by position name |
| `result$params$converged` | logical | Convergence flag |
| `result$params$iterations` | integer | Number of passes to convergence |
| `attr(result, "converged")` | logical | Same as `result$params$converged` |
| `attr(result, "iterations")` | integer | Same as `result$params$iterations` |

The simulator extracts replacement stat values via:

```r
repl_stat <- function(result, position, stat) {
  row <- result$replacement_stats[result$replacement_stats$position == position, ]
  row[[stat]]
}
```

---

## 2. Data Generating Processes (DGPs)

### 2.1 DGP-A: Projection Perturbation (Studies A, B)

Generates synthetic projection data with realistic talent distributions, then adds random noise to simulate year-over-year projection variability.

**Hitter model:**

```
HR[i]  ~ Poisson(mu_HR[i])
R[i]   ~ Poisson(mu_R[i])
RBI[i] ~ Poisson(mu_RBI[i])
SB[i]  ~ Poisson(mu_SB[i])
H[i]   ~ Binomial(AB[i], true_avg[i])
AB[i]  ~ round(Normal(450, 40^2)), clamped to [200, 600]
AVG[i] = H[i] / AB[i]
```

**True mean parameters** (for n=150 hitters, positional breakdown below):

| Position | n_players | mu_HR | mu_R | mu_RBI | mu_SB | true_avg |
|----------|-----------|-------|------|--------|-------|----------|
| C | 15 | 14 | 55 | 55 | 3 | 0.248 |
| 1B | 15 | 24 | 72 | 78 | 5 | 0.265 |
| 2B | 15 | 16 | 72 | 62 | 12 | 0.268 |
| 3B | 15 | 22 | 70 | 75 | 8 | 0.262 |
| SS | 15 | 15 | 70 | 62 | 16 | 0.265 |
| OF | 60 | 20 | 75 | 70 | 14 | 0.268 |
| UTIL | 15 | 18 | 65 | 65 | 8 | 0.260 |

Players are sorted within each position by true talent (highest first) to ensure a realistic talent gradient. The boundary player is at rank `n_teams * roster_slots[pos]` in this sorted order.

**Projection noise (perturbation):**

Each replication draws perturbed projections by adding multiplicative noise:

```
projected_stat[i] = true_stat[i] * (1 + epsilon[i])
epsilon[i] ~ Normal(0, sigma_proj^2)
sigma_proj = 0.10   # 10% coefficient of variation; matches typical projection error
```

Use this perturbation for both "year 1" and "year 2" in Study A's stability comparison.

**Multi-eligibility:** 20% of hitters are randomly assigned a second position (`pos_eligibility` is pipe-delimited). Multi-eligible draws use the empirical distribution from a real NFBC AL-only pool: 2B/SS (30%), 1B/3B (25%), OF/1B (20%), 2B/3B (15%), other (10%).

**Pitcher model:**

```
n_sp = 40, n_rp = 25
IP_SP[i]   ~ Normal(170, 15^2), clamped to [120, 230]
ERA_SP[i]  ~ Normal(3.80, 0.45^2), clamped to [2.50, 6.00]
WHIP_SP[i] ~ Normal(1.22, 0.10^2), clamped to [0.90, 1.80]
W[i]       ~ Poisson(mu_W[i]) where mu_W = IP/9 * 0.44
K[i]       = round(IP[i] * K_per_9[i] / 9) where K_per_9 ~ Normal(8.5, 1.2^2)
SV[i]      = 0  # SP have no saves

IP_RP[i]   ~ Normal(62, 8^2), clamped to [30, 90]
ERA_RP[i]  ~ Normal(3.50, 0.65^2), clamped to [1.50, 7.00]
WHIP_RP[i] ~ Normal(1.22, 0.15^2), clamped to [0.80, 2.00]
SV[i]      ~ Poisson(mu_SV) where mu_SV = 0 for most, 30 for top 3 RP per team
```

Pitchers are sorted by true talent within SP and RP groups. Role column is assigned before noise perturbation.

**Swingmen:** Exactly 4 SP are drawn with `IP ~ Normal(95, 8^2)` (clamp to 80–120) to populate the swingman range. These are included in the pitcher pool.

**Configuration for Studies A and B:**

```r
cfg_study_ab <- league_config(
  n_teams       = 12L,
  roster_slots  = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L, UTIL=1L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories    = c("HR", "R", "RBI", "SB", "AVG", "W", "K", "SV", "ERA", "WHIP"),
  league_type   = "mixed",
  budget        = 260L
)
```

### 2.2 DGP-C: Multi-Eligible Pool (Study C)

Designed to stress-test the multi-position reassignment loop convergence.

**Structure:** 60% of non-catcher, non-OF hitters are multi-eligible (elevated from 20% baseline). This creates maximum churn in the highest-PAR assignment loop.

```
n_hitters = 130
multi_eligible_fraction = 0.60 (applies to 2B, 3B, SS, 1B players)
multi_eligible_pairs: 2B/SS (40%), 1B/3B (35%), 2B/3B (25%)
```

True talent drawn from DGP-A distributions. No noise perturbation (fixed projections) so the loop is the only source of variability. Convergence is a deterministic property for fixed inputs — the simulator verifies that convergence occurs within `max_iter = 25` passes across many random seeds (pool configurations).

**Seed variation:** The pool composition (which players are multi-eligible, their PAR values) varies across replications via the random seed. The DGP generates 500 distinct pool configurations.

### 2.3 DGP-D: Thin AL-Only Pools (Study D)

Two sub-DGPs for the dynamic K cap study:

**Sub-DGP D1 — 12-team AL SS:**
```
n_teams = 12, roster_slots = c(SS = 1L, ...), league_type = "AL"
n_ss_eligible = 20   # deliberately more than 12 to have boundary + band
Talent gradient: SS players sorted by true talent; clear bimodal at rank 12-13
                 (genuine talent gap to test cliff detection interaction)
```

**Sub-DGP D2 — 12-team AL C:**
```
n_teams = 12, roster_slots = c(C = 1L, ...), league_type = "AL"
n_c_eligible = 20
Talent gradient: sharper drop-off after rank 10 for catchers (known empirical pattern)
```

Full configs:

```r
cfg_al_ss <- league_config(
  n_teams      = 12L,
  roster_slots = c(C=1L, `1B`=1L, `2B`=1L, `3B`=1L, SS=1L, OF=3L, DH=1L, UTIL=1L),
  pitcher_slots = c(SP=6L, RP=3L),
  categories   = c("HR", "R", "RBI", "SB", "AVG", "W", "K", "SV", "ERA", "WHIP"),
  league_type  = "AL",
  budget       = 260L
)
# cfg_al_c is identical (same structure; C pool is the thin one)
```

K_eff arithmetic:
- 12-team AL SS: `n_rostered_pos = 12 * 1 = 12`; `K_eff = min(3, floor(12/4)) = min(3, 3) = 3`
- 5-team AL SS (exploratory sub-scenario): `n_rostered_pos = 5`; `K_eff = min(3, floor(5/4)) = min(3, 1) = 1`

The simulator constructs both the 12-team and 5-team configurations to verify the dynamic cap formula numerically.

### 2.4 DGP-E: Rank Invariance (Study E)

**Purpose:** Verify that a fixed-quality pitcher ranks at the same boundary position regardless of league size.

**Structure:**

```
A focal pitcher F has fixed projections: ERA=3.50, WHIP=1.15, IP=175, W=13, K=175
Remaining pitchers: drawn from DGP-A pitcher model with n_sp = {35, 55} to pad to
  (10-team × 6 SP slots) = 60 SP or (15-team × 6 SP slots) = 90 SP plus K=3 buffer

Two league configurations:
  cfg_10team: n_teams=10, pitcher_slots=c(SP=6L, RP=3L)
  cfg_15team: n_teams=15, pitcher_slots=c(SP=6L, RP=3L)
  (Hitter structure identical to DGP-A with matching n_teams scale)

Focal pitcher F is always included at a fixed position in the projections data frame.
```

**Rank metric:**

In each replication, rank all SP by composite score (IP-weighted ERA/WHIP contributions under `boundary_rate_method = "raw_ip"`). Record `rank_10[r]` and `rank_15[r]` for focal pitcher F in replication r. The boundary is at position 60 (10-team) or 90 (15-team) — F's RANK relative to the boundary is what matters.

```
boundary_rank_10 = 60   (n_teams * SP_slots = 10 * 6)
boundary_rank_15 = 90

rank_vs_boundary_10[r] = rank_10[r] - boundary_rank_10
rank_vs_boundary_15[r] = rank_15[r] - boundary_rank_15
```

**Acceptance criterion:** `median(|rank_vs_boundary_10 - rank_vs_boundary_15|)` across 500 replications <= 2. Ranks are relative to the boundary; a fixed-quality pitcher should sit at approximately the same boundary-relative rank regardless of league depth.

---

## 3. Scenario Grid

### Study A: Boundary-Band Stability

| Dimension | Values |
|-----------|--------|
| Band width K | 1, 3 |
| Projection noise sigma | 0.10 |
| n_teams | 12 |
| catcher_adjustment_method | "split_pool" |

Total scenarios: 2 (K=1 vs K=3). Replications per scenario: R = 500.

For each replication r:
1. Draw "Year 1" projections from DGP-A with sigma=0.10, seed = MASTER_SEED + r
2. Draw "Year 2" projections from DGP-A with sigma=0.10, seed = MASTER_SEED + R + r
3. Run `replacement_level()` with each K on Year 1 projections → `repl_1[K]`
4. Run `replacement_level()` with each K on Year 2 projections → `repl_2[K]`
5. Record `delta_HR_1B[K, r] = |repl_1[K]["1B", "HR"] - repl_2[K]["1B", "HR"]|`
6. Also record for SS HR, SP ERA, RP ERA (to cover multiple positions and stat types)

Metrics: `var_delta_HR_1B[K]` across 500 replications; `var_delta_ERA_SP[K]`.

### Study B: Zero-Sum Invariant

| Dimension | Values |
|-----------|--------|
| catcher_adjustment_method | "split_pool", "positional_default", "partial_offset", "none" |

Total scenarios: 4. Replications per scenario: R = 500.

For each replication r and each `catcher_adjustment_method` m:
1. Draw projections from DGP-A, seed = MASTER_SEED + r
2. Run `replacement_level()` with method m
3. Compute zero-sum check value:
   ```r
   pa  <- result$positional_adjustments
   rs  <- config$roster_slots
   zs_pos <- if (m == "split_pool") setdiff(c("C","1B","2B","3B","SS","OF"), "C")
              else c("C","1B","2B","3B","SS","OF")
   zs_pos <- intersect(zs_pos, names(pa))
   check_val <- abs(sum(rs[zs_pos] * pa[zs_pos]))
   ```
4. Record `check_val[m, r]`

Metrics: `max(check_val[m, r])` across all replications and methods. Acceptable: < 1e-6 for all.

### Study C: Multi-Eligible Convergence

| Dimension | Values |
|-----------|--------|
| Pool seed (random pool composition) | 500 distinct seeds |

Total scenarios: 1 pool structure × 500 random seeds. Replications: 500.

For each replication r:
1. Generate DGP-C pool with seed = MASTER_SEED + r
2. Run `replacement_level()` with `multi_pos = "highest_par"`, `max_iter = 25L`
3. Record: `converged[r]`, `iterations[r]`

Metrics: convergence rate (% converged), median iterations, 95th percentile iterations.

### Study D: Dynamic K Cap in Thin Pools

| Dimension | Values |
|-----------|--------|
| League configuration | cfg_al_ss (12-team), cfg_al_c (12-team), cfg_5team_ss (5-team; exploratory) |
| K (default) | 3 |

Total scenarios: 3 configurations. Replications per scenario: R = 500.

For each replication r and each config c:
1. Generate DGP-D pool with seed = MASTER_SEED + r
2. Run `replacement_level()` with `band_width = 3L`
3. Record: `n_band_players[pos, c, r]` from `result$replacement_stats`
4. Compute: `K_eff_observed = (n_band_players - 1) / 2` (back-calculated from band size)

Expected K_eff values:
- 12-team AL SS: `K_eff = 3` → band = 7 (if no cliff truncation)
- 12-team AL C: `K_eff = 3` → band = 7 (if no cliff truncation)
- 5-team SS: `K_eff = 1` → band = 3

Metrics: `K_eff_observed` distribution; proportion of replications where `K_eff_observed <= K_expected`.

### Study E: Rank Invariance

| Dimension | Values |
|-----------|--------|
| League size | 10 teams, 15 teams |

Total scenarios: 2. Replications per scenario: R = 500.

For each replication r and each league size L:
1. Generate DGP-E pitcher pool (with focal pitcher F fixed) with seed = MASTER_SEED + r
2. Run `replacement_level()` with `boundary_rate_method = "raw_ip"`
3. Record focal pitcher F's rank relative to the SP boundary

Metrics: `median(|rank_vs_boundary_10 - rank_vs_boundary_15|)` across 500 replications.

---

## 4. Total Simulation Budget

| Study | Scenarios | Replications | Total Calls |
|-------|-----------|--------------|-------------|
| A | 2 | 500 each | 2,000 (2 calls per replication per K) |
| B | 4 | 500 each | 2,000 |
| C | 1 | 500 | 500 |
| D | 3 | 500 each | 1,500 |
| E | 2 | 500 each | 1,000 |
| **Total** | | | **7,000** |

Justification for R = 500: At R=500 the 95% confidence interval for a 95% coverage rate is ±0.97 percentage points (binomial). For the zero-sum violation study (Study B), the expected violation rate is exactly 0%; a single violation in 500 replications would be detected with probability 1 − (1−1/1e6)^500 ≈ 100%, so R=500 is amply sufficient. For variance comparisons (Study A), R=500 achieves SE of variance estimate ≈ σ²*sqrt(2/(R-1)) ≈ 5% of the true variance, sufficient to detect a 2× difference in K=1 vs K=3 variance.

---

## 5. Seed Management

```r
MASTER_SEED     <- 20260416L    # fixed; from request date
set.seed(MASTER_SEED)

# Per-replication seeds:
scenario_seed <- function(study, scenario_idx, replication) {
  MASTER_SEED + study * 100000L + scenario_idx * 10000L + replication
}
```

Per-replication RNG: `set.seed(scenario_seed(study, scenario_idx, r))` before each DGP draw. Use base R's default Mersenne-Twister RNG.

---

## 6. Performance Metrics

### Study A Metrics

| Metric | Definition | Unit |
|--------|-----------|------|
| `var_K3_HR_1B` | Variance of year-over-year delta in replacement HR for 1B across 500 replications, K=3 | HR² |
| `var_K1_HR_1B` | Same for K=1 | HR² |
| `var_K3_ERA_SP` | Variance of YoY delta in SP replacement ERA, K=3 | ERA² |
| `var_K1_ERA_SP` | Same for K=1 | ERA² |
| `var_ratio_HR_1B` | `var_K3_HR_1B / var_K1_HR_1B` | dimensionless |
| `var_ratio_ERA_SP` | `var_K3_ERA_SP / var_K1_ERA_SP` | dimensionless |

### Study B Metrics

| Metric | Definition | Unit |
|--------|-----------|------|
| `max_zero_sum_violation[m]` | Max of `check_val` across 500 replications for method m | unitless |
| `n_violations[m]` | Count of replications where `check_val >= 1e-6` | integer |

### Study C Metrics

| Metric | Definition |
|--------|-----------|
| `convergence_rate` | % of 500 replications where `converged = TRUE` |
| `median_iterations` | Median of `iterations` across replications |
| `p95_iterations` | 95th percentile of `iterations` |
| `n_max_iter_hits` | Count of replications that hit `max_iter = 25` |

### Study D Metrics

| Metric | Definition |
|--------|-----------|
| `K_eff_12_SS` | Mean observed `K_eff` for SS in 12-team AL config |
| `K_eff_12_C` | Mean observed `K_eff` for C in 12-team AL config |
| `K_eff_5_SS` | Mean observed `K_eff` for SS in 5-team config |
| `pct_correct_K_eff` | % replications where `K_eff_observed == K_eff_expected` |

### Study E Metrics

| Metric | Definition |
|--------|-----------|
| `median_abs_rank_diff` | `median(|rank_vs_boundary_10 - rank_vs_boundary_15|)` |
| `p90_abs_rank_diff` | 90th percentile of `|rank_vs_boundary_10 - rank_vs_boundary_15|` |
| `pct_within_2` | % replications where `|rank_vs_boundary_10 - rank_vs_boundary_15| <= 2` |

---

## 7. Acceptance Criteria

The simulator reports these metrics to the tester (via `simulation.md` in the run directory). Tester asserts pass/fail against each threshold.

| Study | Metric | Threshold | Pass condition |
|-------|--------|-----------|----------------|
| A | `var_ratio_HR_1B` | < 1.0 | K=3 has strictly lower year-over-year variance than K=1 |
| A | `var_ratio_ERA_SP` | < 1.0 | K=3 has strictly lower variance than K=1 for SP ERA |
| B | `n_violations[split_pool]` | == 0 | Zero zero-sum violations for default method |
| B | `n_violations[positional_default]` | == 0 | Zero violations |
| B | `n_violations[partial_offset]` | == 0 | Zero violations |
| B | `n_violations[none]` | == 0 | Zero violations |
| B | `max_zero_sum_violation[*]` | < 1e-6 | All violations < tolerance |
| C | `convergence_rate` | >= 0.99 | >= 99% of replications converge |
| C | `median_iterations` | <= 5 | Median <= 5 passes when converged |
| C | `n_max_iter_hits` | <= 5 | Fewer than 1% of 500 replications hit max_iter |
| D | `K_eff_12_SS` (mean) | == 3.0 | Dynamic cap = 3 for 12-team SS in all replications |
| D | `K_eff_12_C` (mean) | == 3.0 | Dynamic cap = 3 for 12-team C in all replications |
| D | `K_eff_5_SS` (mean) | == 1.0 | Dynamic cap = 1 for 5-team SS in all replications |
| D | `pct_correct_K_eff` | == 1.0 | 100% correct across all configurations |
| E | `median_abs_rank_diff` | <= 2.0 | Focal pitcher ranks within ±2 of boundary-relative rank |
| E | `pct_within_2` | >= 0.90 | >= 90% of replications within ±2 |

Note on Study D: K_eff is a deterministic function of n_rostered_pos and K — it has no randomness. The metric should always be exactly correct. If any replication produces an incorrect K_eff, that indicates a computation bug (not sampling variation). `pct_correct_K_eff = 1.0` is a hard requirement.

---

## 8. Output Format

Simulator writes `simulation.md` to the run directory with the following structure:

```markdown
# Simulation Results: replacement_level()

Request ID: replacement-2026-04-16
Date: <run date>
R replications per scenario: 500
Master seed: 20260416

## Study A: Boundary-Band Stability

| Metric | K=1 | K=3 | Ratio (K3/K1) | Pass? |
|--------|-----|-----|---------------|-------|
| var_HR_1B  | ... | ... | ... | PASS/FAIL |
| var_ERA_SP | ... | ... | ... | PASS/FAIL |

## Study B: Zero-Sum Invariant

| catcher_method | max_violation | n_violations | Pass? |
|----------------|---------------|--------------|-------|
| split_pool        | ... | 0 | PASS/FAIL |
| positional_default| ... | 0 | PASS/FAIL |
| partial_offset    | ... | 0 | PASS/FAIL |
| none              | ... | 0 | PASS/FAIL |

## Study C: Multi-Eligible Convergence

| Metric | Value | Threshold | Pass? |
|--------|-------|-----------|-------|
| convergence_rate   | ... | >= 0.99 | PASS/FAIL |
| median_iterations  | ... | <= 5    | PASS/FAIL |
| n_max_iter_hits    | ... | <= 5    | PASS/FAIL |

## Study D: Dynamic K Cap

| Config | K_eff_expected | K_eff_observed (mean) | pct_correct | Pass? |
|--------|----------------|-----------------------|-------------|-------|
| 12-team AL SS | 3 | ... | ... | PASS/FAIL |
| 12-team AL C  | 3 | ... | ... | PASS/FAIL |
| 5-team SS     | 1 | ... | ... | PASS/FAIL |

## Study E: Rank Invariance

| Metric | Value | Threshold | Pass? |
|--------|-------|-----------|-------|
| median_abs_rank_diff | ... | <= 2.0 | PASS/FAIL |
| pct_within_2         | ... | >= 0.90 | PASS/FAIL |

## Overall: PASS / FAIL

<n_pass> / <n_total> criteria passed.
```

Also write a CSV or RDS file of per-replication raw results to `simulation_results.rds` in the run directory for tester reproducibility verification.

---

## 9. Simulation Script Location

Write the simulation harness to:

```
inst/simulations/replacement-mc.R
```

The script must be self-contained: it sources or loads `replacement_level()` from the package (via `devtools::load_all()`), runs all five studies, and writes `simulation.md` and `simulation_results.rds` to the run directory path specified by an environment variable `STATSCLAW_RUN_DIR` (default: the run directory path for this request).

---

## 10. Reproducibility Requirements

- All DGP draws use base R's Mersenne-Twister RNG with the seed strategy from §5
- The simulation script must be re-runnable and produce identical results when re-executed with the same seeds
- Parallelization is permitted (via `parallel::mclapply()`) but the seed strategy must ensure per-replication determinism regardless of execution order
- The script must complete in under 15 minutes on a modern laptop (4 cores, 16GB RAM). Estimated runtime: 7000 calls × ~5ms/call = ~35 seconds. Add 10× safety margin for R startup overhead = ~6 minutes. Well within budget.

---

## 11. Notes on Study Design Rationale

**Study A (variance ratio < 1.0):** The theoretical argument is that averaging over more players (K=3 band vs K=1 single player) reduces sensitivity to individual projection outliers. The test verifies the empirical variance reduction, not just that the band is wider.

**Study B (zero-sum exactly 0 to 1e-6):** This is a deterministic algebraic identity, not a statistical expectation. Any violation indicates a computation bug. The Monte Carlo structure here is across different projection inputs (not a distributional test). R=500 ensures the result holds across a diverse set of inputs.

**Study C (convergence >= 99%):** Non-convergence at max_iter is a known acceptable outcome (emits a warning, not an error). But for typical synthetic data the loop should converge. Setting threshold at 99% (5 allowed non-convergences out of 500) allows for edge cases without causing a false failure.

**Study D (K_eff exactly correct):** `K_eff = min(K, floor(n_rostered_pos / 4))` is a closed-form formula. The Monte Carlo here checks that the implementation uses this formula correctly across varied projection draws (the K_eff value should be deterministic for fixed n_rostered_pos and K, regardless of the actual projections).

**Study E (rank invariance):** The `raw_ip` rate method is specifically designed to be league-depth-independent. A fixed-quality pitcher's rank should shift by at most ±2 positions (boundary-relative) between a 10-team and 15-team league, because the only thing that changes is how many pitchers sit above and below — not the pitcher's own contribution.
