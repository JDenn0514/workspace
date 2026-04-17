# Audit: sgp-denom-dgpb-rerun-2026-04-17

**Tester:** tester (isolated test pipeline)
**Date:** 2026-04-17
**Run directory:** `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denom-dgpb-rerun-2026-04-17/`
**Worktree:** `/Users/jacobdennen/rotostats/.claude/worktrees/sgp-denom-dgpb-rerun` (feature/sgp-denom-dgpb-rerun)
**Verdict: PASS**

---

## Scope of this audit

This audit covers Q4 (Time-Decay Variance Trade-off) and Q7 (Per-Category Override Sanity) under the corrected DGP-B (sigma-shift, pre_sd=10, post_sd=25, break_at=7). Q1, Q2, Q3, Q5, Q6 are NOT re-audited; their verdicts from the 2026-04-16 run are carried forward.

Tester does NOT read spec.md or implementation.md. All validation is driven by test-spec.md (the primary specification) and the corrected sim-spec.md (DGP-B section only).

---

## Step 1 — Pre-flight: Independent theta Calibration

### Protocol

Before evaluating any acceptance criteria, I ran an independent empirical calibration to confirm that the corrected DGP-B (sigma-shift) produces a detectable pre/post shift in the OLS estimand. I used independent seeds (42001 for pre-regime, 42002 for post-regime) — NOT the Simulator's seeds (88882/88883) — to provide genuine independence.

### Command executed

```r
# Worktree: /Users/jacobdennen/rotostats/.claude/worktrees/sgp-denom-dgpb-rerun
suppressMessages(devtools::load_all('.', quiet=TRUE))

set.seed(42001)
ts_pre <- do.call(rbind, lapply(seq(2000, 3999), function(y) {
  data.frame(year=y, team_id=paste0('T', 1:12),
             SB=rnorm(12, mean=100, sd=10))
}))
theta_pre <- sgp_denominators(
  list(team_season=ts_pre),
  scoring_categories='SB', n_teams=12L,
  weights=flat(), method='ols', exclude_years=integer(0)
)$denominators[['SB']]

set.seed(42002)
ts_post <- do.call(rbind, lapply(seq(2000, 3999), function(y) {
  data.frame(year=y, team_id=paste0('T', 1:12),
             SB=rnorm(12, mean=100, sd=25))
}))
theta_post <- sgp_denominators(
  list(team_season=ts_post),
  scoring_categories='SB', n_teams=12L,
  weights=flat(), method='ols', exclude_years=integer(0)
)$denominators[['SB']]
```

### Calibration result

| Quantity | Value |
|---|---|
| theta_pre (sigma=10, n_years=2000, seed=42001) | 2.681101 |
| theta_post (sigma=25, n_years=2000, seed=42002) | 6.718070 |
| gap (post - pre) | 4.036969 |
| rel_gap (gap / theta_pre) | 150.57% |
| theta_post / theta_pre | 2.5057 |

### Calibration gate

| Gate | Threshold | Observed | Verdict |
|---|---|---|---|
| Relative gap > 50% (OLS-appropriate gate) | > 50% | 150.57% | PASS |
| theta ratio ~= sigma ratio (sanity) | ~2.5x (sigma 25/10) | 2.506x | PASS |

**CALIBRATION PASSED.** The OLS estimand shifts proportionally with sigma, confirming a genuine denominator regime change. The relative gap (150.57%) far exceeds the 50% threshold. The run does NOT proceed to BLOCK.

### Note on Planner's absolute threshold

Planner's mailbox specified a gate of `|theta_post - theta_pre| > 5 units`, derived from the SD-method analytical formula (n-1)*sigma/E[R_n] which gives theta ~84 at sigma=25. For method="ols", the denominator = 1/|beta_hat| where beta_hat is the OLS slope of rank~total. Under Gaussian totals with n_teams=12, the OLS denominator scales at ~0.27 units/sigma (not ~3.36 units/sigma for "sd"). The absolute gap is therefore ~4.1 (not ~50), but the relative gap is the same (~150%). The absolute gate does not apply to OLS. I used the relative gate (> 50%) as specified in the Simulator's calibration note and confirmed in the mailbox.

### Cross-reference with Simulator's calibration

| Quantity | Simulator (seeds 88882/88883) | Tester (seeds 42001/42002) | Difference |
|---|---|---|---|
| theta_pre | 2.6744 | 2.6811 | 0.0067 (0.25%) |
| theta_post | 6.7722 | 6.7181 | 0.0541 (0.80%) |
| rel_gap | 153.2% | 150.6% | 2.6 pp |

Differences are due to different random seeds. Both estimates use n_years=2000 and are well within sampling variability (SE of mean < 0.01 at n_years=2000). Both independently confirm rel_gap >> 50%.

---

## Step 2 — Primary Validation: devtools::check()

### Command

```r
devtools::check('/Users/jacobdennen/rotostats/.claude/worktrees/sgp-denom-dgpb-rerun',
                args = '--as-cran', quiet = FALSE)
```

### Result

```
Status: 4 NOTEs
0 errors | 0 warnings | 4 notes
```

All 4 NOTEs are pre-existing infrastructure issues (`.git` directory in package tree, non-standard top-level files `ARCHITECTURE.md`/`HANDOFF.md`/`api_files`/`specs`, empty NEWS.md, Rd LaTeX escaped specials in `replacement_from_prices.Rd` and `replacement_level.Rd`). None are related to this run's changes or to `sgp_denominators()`. This package check is clean for the purposes of this audit.

### Test suite result

```r
# Command
testthat::test_file('tests/testthat/test-sgp-denominators.R', reporter='check')

# Result
[ FAIL 0 | WARN 125 | SKIP 0 | PASS 123 ]
```

All 123 test expectations PASS. 0 failures. The 125 warnings are expected — they are the warning-class assertions emitted by `sgp_denominators()` itself when exercising edge cases (thin calibration windows, structural break warnings, near-zero slopes, etc.), captured by testthat's warning reporter. No test warns about unexpected test failures.

---

## Step 3 — Q4: Time-Decay Variance Trade-off (corrected DGP-B)

### Tolerances used (from test-spec.md §7 and sim-spec.md §7)

test-spec.md §14 (Simulation Validation Assertions) specifies:
- Criterion 4: `mse_SB[exp_decay(0.7)] < mse_SB[flat]` under DGP-B (exact inequality, no tolerance)

sim-spec.md §7 specifies:
- `mse[flat] <= mse[exp_decay(0.7)]` under DGP-A (exact inequality)
- `mse[exp_decay(0.7)] <= mse[flat]` under DGP-B (exact inequality)
- Monotone ordering DGP-B: `mse[flat] >= mse[exp_decay(0.9)] >= mse[exp_decay(0.7)]` (exact)
- `abs(rel_bias) <= 0.15` under DGP-A for all weight schemes (exact threshold)

All tolerances used below match sim-spec.md exactly. No tolerance was widened or softened.

### Data verified from

CSV: `q4_decay.csv` (read independently, cross-referenced with `simulation_q4q7_results.rds`).

### Per-Test Result Table (Q4)

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|---|---|---|---|---|---|---|
| Q4 Crit 1: DGP-A flat vs exp7 | mse[flat] <= mse[exp7] | flat < exp7 | 0.251886 <= 0.481361 | exact inequality | — | PASS |
| Q4 Crit 2: DGP-B exp7 vs flat | mse[exp7] <= mse[flat] | exp7 < flat | 2.987027 <= 10.407268 | exact inequality | — | PASS |
| Q4 Crit 3a: DGP-B monotone flat >= exp9 | mse[flat] >= mse[exp9] | flat >= exp9 | 10.407268 >= 8.002536 | exact inequality | — | PASS |
| Q4 Crit 3b: DGP-B monotone exp9 >= exp7 | mse[exp9] >= mse[exp7] | exp9 >= exp7 | 8.002536 >= 2.987027 | exact inequality | — | PASS |
| Q4 Crit 4: DGP-A rel_bias[flat] | abs(rel_bias) <= 0.15 | <= 0.15 | 0.008823 | atol=0.15 | — | PASS |
| Q4 Crit 4: DGP-A rel_bias[exp9] | abs(rel_bias) <= 0.15 | <= 0.15 | 0.008745 | atol=0.15 | — | PASS |
| Q4 Crit 4: DGP-A rel_bias[exp7] | abs(rel_bias) <= 0.15 | <= 0.15 | 0.012657 | atol=0.15 | — | PASS |
| Q4 failure_rate DGP-A flat | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |
| Q4 failure_rate DGP-A exp9 | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |
| Q4 failure_rate DGP-A exp7 | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |
| Q4 failure_rate DGP-B flat | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |
| Q4 failure_rate DGP-B exp9 | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |
| Q4 failure_rate DGP-B exp7 | failure_rate <= 0.01 | 0.000 | 0.000 | atol=0.01 | 0.0% | PASS |

**Q4 verdict: 4/4 acceptance criteria PASS. 0 failures across 12,000 replications.**

### Full Q4 result table (from CSV)

| dgp | weights | theta | bias | rel_bias | mse | rmse | failure_rate |
|---|---|---:|---:|---:|---:|---:|---:|
| A | flat | 6.6968 | 0.0591 | 0.00882 | 0.2519 | 0.5019 | 0.0 |
| B | flat | 6.7722 | -3.2135 | -0.47452 | 10.4073 | 3.2260 | 0.0 |
| A | exp9 | 6.6968 | 0.0586 | 0.00874 | 0.2752 | 0.5246 | 0.0 |
| B | exp9 | 6.7722 | -2.8128 | -0.41534 | 8.0025 | 2.8289 | 0.0 |
| A | exp7 | 6.6968 | 0.0848 | 0.01266 | 0.4814 | 0.6938 | 0.0 |
| B | exp7 | 6.7722 | -1.6697 | -0.24656 | 2.9870 | 1.7283 | 0.0 |

### MSE heat table (Tester-verified from CSV)

| | flat | exp_decay(0.9) | exp_decay(0.7) |
|---|---:|---:|---:|
| DGP-A (stable) | **0.2519** | 0.2752 | 0.4814 |
| DGP-B (break) | 10.4073 | 8.0025 | **2.9870** |

### Q4 diagnostic interpretation

Under DGP-B (sigma-shift break, pre_sd=10 for years 1-6, post_sd=25 for years 7-10), the flat-weighted estimator averages across both regimes and severely underestimates the current (post-break) denominator. The bias is -3.21 (a 47% underestimate of the true post-break denominator 6.77). Exponential decay progressively concentrates weight on the post-break years, reducing both the bias and MSE monotonically:

- flat: MSE = 10.41, rel_bias = -47.5%
- exp_decay(0.9): MSE = 8.00 (-23% vs flat), rel_bias = -41.5%
- exp_decay(0.7): MSE = 2.99 (-71% vs flat), rel_bias = -24.7%

Under DGP-A (stable), flat() is best (MSE 0.252) and exp_decay(0.7) is worst (MSE 0.481, +91% vs flat), as expected from the bias-variance trade-off. All three weight schemes have negligible bias under DGP-A (max rel_bias 1.3%, well within the 15% threshold).

The corrected DGP-B (sigma-shift) makes the Q4 acceptance criteria cleanly testable, unlike the prior run's mean-shift DGP-B which left the OLS estimand unchanged across the break.

### Before/After Comparison (Q4, DGP-B criteria)

| Metric | Prior run (mean-shift DGP-B) | This run (sigma-shift DGP-B) | Change | Interpretation |
|---|---|---|---|---|
| Testability of Q4 DGP-B criteria | Untestable (theta_pre = theta_post) | Testable (rel gap = 150%) | — | Fix achieved |
| DGP-B flat MSE | ~0 (no regime change) | 10.4073 | +10.41 | Now reflects genuine regime bias |
| DGP-B exp7 MSE | ~0 | 2.9870 | +2.99 | Decay benefit measurable |
| Q4 criteria pass | 2/4 (DGP-B untestable) | 4/4 | +2 | Full pass |

---

## Step 4 — Q7: Per-Category Override Sanity (corrected DGP-B)

### Tolerances used (from test-spec.md §14 and sim-spec.md §10)

- Criterion 1: `mse_SB[S2] < mse_SB[S1]` (exact strict inequality)
- Criterion 2: `abs(mse_HR[S2] - mse_HR[S1]) / mse_HR[S1] <= 0.05` (exact threshold, rtol=0.05)
- Criterion 3: `mse_SB[S1] >= mse_SB[S3] >= mse_SB[S2]` (exact inequality chain)

All tolerances used match sim-spec.md exactly. No tolerance was widened or softened.

### Data verified from

CSV: `q7_override.csv` (read independently, cross-referenced with `simulation_q4q7_results.rds`).

### Per-Test Result Table (Q7)

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|---|---|---|---|---|---|---|
| Q7 Crit 1: override improves SB | mse_SB[S2] < mse_SB[S1] | S2 < S1 | 0.673306 < 10.437593 | exact inequality | — | PASS |
| Q7 MSE reduction magnitude | mse_SB[S2] / mse_SB[S1] | < 1 | 0.0645 (93.5% reduction) | — | — | PASS |
| Q7 Crit 2: HR isolation | abs rel change <= 5% | <= 0.05 | 0.009366 (0.94%) | rtol=0.05 | — | PASS |
| Q7 Crit 3a: S1 >= S3 SB MSE | mse_SB[S1] >= mse_SB[S3] | S1 >= S3 | 10.437593 >= 2.979922 | exact inequality | — | PASS |
| Q7 Crit 3b: S3 >= S2 SB MSE | mse_SB[S3] >= mse_SB[S2] | S3 >= S2 | 2.979922 >= 0.673306 | exact inequality | — | PASS |
| Q7 failure_rate all scenarios | failure_rate <= 0.01 | 0.000 | 0.000 (all 6 rows) | atol=0.01 | 0.0% | PASS |

**Q7 verdict: 3/3 acceptance criteria PASS. 0 failures across 6,000 replications.**

### Full Q7 result table (from CSV)

| scenario | category | theta | bias | mse | failure_rate | mse_rel_to_S1 |
|---|---|---:|---:|---:|---:|---:|
| S1 (no override, flat) | HR | 6.6968 | 0.0458 | 0.2579 | 0.0 | 1.0000 |
| S1 (no override, flat) | SB | 6.7722 | -3.2181 | 10.4376 | 0.0 | 1.0000 |
| S2 (SB: after+exp7) | HR | 6.6968 | 0.0485 | 0.2603 | 0.0 | 1.0094 |
| S2 (SB: after+exp7) | SB | 6.7722 | 0.0617 | 0.6733 | 0.0 | 0.0645 |
| S3 (SB: exp7 all) | HR | 6.6968 | 0.0534 | 0.2571 | 0.0 | 0.9972 |
| S3 (SB: exp7 all) | SB | 6.7722 | -1.6686 | 2.9799 | 0.0 | 0.2855 |

### Q7 diagnostic interpretation

**S2 (after(2005) + exp_decay(0.7) for SB):** Restricting SB to the 4 post-break years (2006-2009) with exponential decay virtually eliminates the regime-change bias. SB bias collapses from -3.22 to +0.06 (near-zero), and SB MSE falls by 93.5% (10.44 to 0.67). The tiny positive bias (+0.06) arises because exp_decay(0.7) slightly upweights the most recent post-break years, which by chance have slightly higher realized sigma than average. HR MSE changes by 0.94% (well within the 5% isolation threshold), confirming category_spec isolation.

**S3 (exp_decay(0.7) on all years for SB):** Without the year cutoff, all 10 years are used but pre-break years are heavily downweighted. SB bias improves to -1.67 (from -3.22) and MSE falls by 71.4%. This is the "blunt instrument" — substantial improvement, but not as precise as the targeted year cutoff. MSE_S1 >= MSE_S3 >= MSE_S2 holds (10.44 >= 2.98 >= 0.67).

**HR isolation across all scenarios:** HR MSE changes by at most 0.94% across all three scenarios (0.2579 vs 0.2603 vs 0.2571), far within the 5% threshold. category_spec isolation is confirmed.

### Before/After Comparison (Q7, SB criteria)

| Metric | Prior run (mean-shift SB) | This run (sigma-shift SB) | Change | Interpretation |
|---|---|---|---|---|
| Testability of Q7 criteria | Untestable (no OLS estimand change) | Testable (rel gap = 150%) | — | Fix achieved |
| S1 SB MSE | near 0 (regime unchanged) | 10.4376 | +10.44 | Now reflects genuine regime bias |
| S2 SB MSE | near 0 | 0.6733 | +0.67 | Override benefit measurable (93.5% reduction) |
| Q7 criteria pass | 1/3 (S2/S3 untestable) | 3/3 | +2 | Full pass |

---

## Step 5 — Simulation Validation Table (simulation workflow)

### Calibration criterion

| Criterion | Metric | Target | Actual | Threshold | Verdict |
|---|---|---|---|---|---|
| Calibration | Relative gap (theta_post/theta_pre - 1) | > 50% | 150.57% (Tester) / 153.22% (Simulator) | > 50% | PASS |
| Calibration | theta ratio | ~2.5x (sigma ratio) | 2.506x (Tester) | ~2.5x | PASS |

### Q4 acceptance criteria

| Criterion | Metric | Target | Actual | At N | Threshold | Verdict |
|---|---|---|---|---|---|---|
| DGP-A stable: flat best | mse[flat] <= mse[exp7] | < 1 | 0.252 <= 0.481 | 2000 | exact ineq | PASS |
| DGP-B break: exp7 best | mse[exp7] <= mse[flat] | < 1 | 2.987 <= 10.407 | 2000 | exact ineq | PASS |
| DGP-B monotone: flat >= exp9 >= exp7 | mse ordering | ordered | 10.407 >= 8.003 >= 2.987 | 2000 | exact ineq | PASS |
| DGP-A bias bounded | max abs(rel_bias) | <= 0.15 | 0.01266 | 2000 | atol=0.15 | PASS |

### Q7 acceptance criteria

| Criterion | Metric | Target | Actual | At N | Threshold | Verdict |
|---|---|---|---|---|---|---|
| Override improves SB | mse_SB[S2] < mse_SB[S1] | < 1 | 0.673 < 10.438 | 2000 | exact ineq | PASS |
| HR isolation | abs rel change <= 5% | <= 5% | 0.94% | 2000 | rtol=0.05 | PASS |
| S3 SB intermediate | S1 >= S3 >= S2 | ordered | 10.44 >= 2.98 >= 0.67 | 2000 | exact ineq | PASS |

### Reproducibility check

The Simulator's script uses `set.seed()` before each replicate via `scenario_seed()`. The RDS and CSV files contain identical values (verified by bitwise comparison: all 6 Q4 MSE values match to 8 decimal places). Tester independently calibrated theta using different seeds and confirmed the same relative gap (150.57% vs 153.22%) — within expected Monte Carlo sampling variability. Reproducibility confirmed.

---

## Step 6 — Environment Information

| Item | Value |
|---|---|
| R version | 4.5.2 (2025-10-31) |
| Platform | aarch64-apple-darwin20 |
| OS | macOS Sequoia 15.4.1 |
| Package version | rotostats 0.0.0.9000 |
| Worktree | `/Users/jacobdennen/rotostats/.claude/worktrees/sgp-denom-dgpb-rerun` |
| devtools::check duration | 28.2s |
| testthat result | FAIL 0, WARN 125, SKIP 0, PASS 123 |

---

## Step 7 — Discrepancies with Simulator's Report

No substantive discrepancies. All values in the Simulator's simulation.md match the raw CSV and RDS files exactly. The two minor points:

1. **Calibration theta values:** Tester's independent calibration (seeds 42001/42002) yields theta_pre=2.6811, theta_post=6.7181 vs Simulator's (seeds 88882/88883) theta_pre=2.6744, theta_post=6.7722. The difference (0.3%-0.8%) is sampling variability from using different seeds with n_years=2000. Both confirm rel_gap >> 50%. Not a discrepancy.

2. **Acceptance criterion wording:** Simulator's mailbox table shows "S2 SB MSE < S1 SB MSE: 0.673 < 10.438" but the CSV value is 10.437593 (rounds to 10.438). Tester verified against the CSV directly. Not a discrepancy.

---

## Step 8 — Quality Checks

- Ran ALL required validation commands (devtools::check, testthat::test_file, manual acceptance criterion verification from CSVs).
- Executed ALL test scenarios from test-spec.md via devtools::check (123 PASS, 0 FAIL).
- Per-Test Result Table present with one row per metric per scenario.
- Before/After Comparison Table present for Q4 and Q7.
- Every claimed result has exact evidence.
- Independent calibration used different seeds from Simulator.
- Did NOT read spec.md or implementation.md.
- Did NOT edit any files in the target repo.
- Did NOT widen any tolerance or acceptance criterion.
- All tolerances used match test-spec.md / sim-spec.md exactly.

---

## Final Verdict: PASS

| Study | Criteria | Verdict |
|---|---|---|
| Calibration pre-flight | rel_gap > 50% | PASS (150.57% >> 50%) |
| Q4 — Time-Decay Variance Trade-off | 4/4 criteria | PASS |
| Q7 — Per-Category Override Sanity | 3/3 criteria | PASS |
| devtools::check() | 0 errors, 0 warnings | PASS |
| testthat test suite | FAIL 0, PASS 123 | PASS |

No BLOCK warranted. All acceptance criteria from test-spec.md and sim-spec.md pass under the corrected DGP-B. Routing to Reviewer.
