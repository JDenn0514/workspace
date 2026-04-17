# Simulation Report: `sgp_denominators()` Q4 + Q7 Re-run Under Corrected DGP-B

> **Pipeline:** Simulation pipeline — Q4 and Q7 only
> **Run ID:** sgp-denom-dgpb-rerun-2026-04-17
> **Date:** 2026-04-17
> **Total wall time:** 8.6 min (Q4 + Q7 only)
> **Total runs:** 18,000 (Q4: 12,000; Q7: 6,000; 0 failures)
> **Engine:** Base R Mersenne-Twister, master seed `2026041601`, same `scenario_seed()` as 2026-04-16
> **Black-box interface:** public `sgp_denominators()` API only (`flat()`, `exp_decay()`, `cal()`, `cal_spec()`, `after()`)

## Scope of this run

This run re-executes ONLY Q4 (Time-Decay Variance Trade-off) and Q7 (Per-Category Override
Sanity) under the corrected DGP-B, which shifts within-year σ across the structural break
(pre_sd=10, post_sd=25, break_at=7) instead of the league mean. Q1, Q2, Q3, Q5, Q6 are NOT
re-run; their verdicts from the 2026-04-16 run are referenced by path below.

---

## 0. Prior-Run Sections (Unchanged)

The following studies from the 2026-04-16 run are carried forward verbatim. Their verdicts,
code, and result tables are authoritative and unaffected by the DGP-B correction.

| Study | Verdict | Prior-run CSV |
|---|---|---|
| Q1 — OLS Convergence | ✅ All 4 criteria pass | `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q1_convergence.csv` |
| Q2 — Finite-Sample Bias Direction | ✅ All 3 criteria pass (Jensen confirmed, rel_bias ≈ 1%) | `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q2_bias.csv` |
| Q3 — Bootstrap CI Coverage | ⚠️ Coverage 0.847 (borderline vs 0.85 floor; attributable to Jensen bias) | `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q3_coverage.csv` |
| Q5 — Expected-Range Anchor Verification | ✅ Max abs error 3.7e-4 < 1e-3 | `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q5_anchors.csv` |
| Q6 — Method Comparison | ✅ All 4 criteria pass | `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q6_methods.csv`, `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q6_methods_heavy.csv` |

Full narrative, tables, and diagnostics for these studies: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/simulation.md`

---

## 0.1 Estimand Calibration Note (carried forward and updated)

The sim-spec's analytical formula `θ_OLS ≈ (n-1)·σ / E[R_n]` applies to `method = "sd"`,
not to `method = "ols"`. The OLS estimator returns `1/|β̂|` where β̂ is the slope of
`standings_rank ~ total`. Under Gaussian totals, `denom_OLS ∝ σ` at a rate of approximately
0.27 units per unit σ for n_teams=12 — not the SD-formula rate of 3.36/unit.

This was first documented in the 2026-04-16 run (§0.2). All simulations in both runs use
empirical θ calibration (2000-year single call to `sgp_denominators()`) rather than the
analytical formula. Under the corrected DGP-B (σ = {10, 25}):

| Regime | σ | θ_OLS (empirical) | θ_SD (analytical) |
|---|---|---|---|
| Pre-break | 10 | 2.6744 | 33.76 |
| Post-break | 25 | 6.7722 | 84.40 |
| Gap | — | 4.0978 | 50.64 |
| Relative gap | — | 153.2% | 150.0% |

The relative gap (≈150%) is consistent across both methods. The OLS absolute denominator
is on a different scale but the proportional shift is the same, confirming the DGP-B σ-shift
creates a genuine, proportionally equivalent regime change for the OLS estimator.

### Calibration check (gate for Tester)

Before running Q4/Q7 acceptance criteria, the DGP-B σ-shift was verified to produce a
detectable pre/post θ shift:

```
theta_pre  (pure σ=10 regime, n_years=2000) = 2.6744
theta_post (pure σ=25 regime, n_years=2000) = 6.7722
Gap = 4.0978
Gap / theta_pre = 1.532 (153.2%)
SE of mean theta at R=2000 ≈ 0.006
z-score = 4.10 / 0.006 ≈ 683 >> 5 threshold
```

**Calibration PASSED.** The σ-shift DGP-B creates a genuine denominator regime change in
the OLS estimator, validly testable at R=2000.

---

## 1. Q1 — OLS Convergence

**Verdict: ✅ PASS (4/4 criteria)**

Results unchanged from 2026-04-16 run. See prior simulation.md §1 and
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q1_convergence.csv`

Summary: rel_bias = 0.0068 at n_years=20 (≤0.10), 0.0037 at n_years=50 (≤0.05). Log-log
RMSE slope = −0.490 (≤−0.40). Failure rate = 0.000 everywhere.

---

## 2. Q2 — Finite-Sample Bias Direction

**Verdict: ✅ PASS (3/3 criteria)**

Results unchanged from 2026-04-16 run. See prior simulation.md §2 and
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q2_bias.csv`

Summary: Bias direction confirmed positive in all 9 scenarios. rel_bias ≈ 1% at (n=12,
n_years=6), within the ≤0.25 tolerance. Jensen gap = 0.064 (0.96% of θ).

---

## 3. Q3 — Bootstrap CI Coverage

**Verdict: ⚠️ BORDERLINE (coverage 0.847 at nominal 95%)**

Results unchanged from 2026-04-16 run. See prior simulation.md §3 and
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q3_coverage.csv`

Summary: Coverage = 0.847 at nominal 95%; fails tight [0.93, 0.97] and nominal [0.90, 1.00]
bands; borderline on the ≥0.85 floor (within binomial sampling error). Undercoverage
attributable to Jensen's inequality bias (documented Q2). BCa bootstrap deferred.

---

## 4. Q4 — Time-Decay Variance Trade-off (FRESH — corrected DGP-B)

**Setup:** 2 DGPs × 3 weight schemes, R = 2000 per cell, n_years = 10, n_teams = 12.

- **DGP-A:** stable Gaussian, σ = 25, θ_A = 6.6968 (empirical calibration, matches prior run).
- **DGP-B (corrected):** σ-shift break at year index 7. Pre-break σ = 10 (years 1–6),
  post-break σ = 25 (years 7–10). Mean = 100 throughout. Target θ_B = 6.7722 (post-break
  regime, empirically calibrated from 2000-year pure post-regime run).

**Correction from prior run:** The 2026-04-16 DGP-B shifted the league mean (100 → 140)
while holding σ = 20 fixed. The OLS denominator is mean-invariant, so no denominator
regime change occurred and the criteria were untestable. The corrected DGP-B shifts σ,
producing a 2.5× spread change and a 153% relative shift in the OLS denominator.

### Results table

| dgp | weights | θ | bias | rel_bias | mse | rmse | failure% |
|---|---|---:|---:|---:|---:|---:|---:|
| A | flat | 6.697 | 0.0591 | 0.00882 | **0.2519** | 0.502 | 0.0 |
| A | exp_decay(0.9) | 6.697 | 0.0586 | 0.00874 | 0.2752 | 0.525 | 0.0 |
| A | exp_decay(0.7) | 6.697 | 0.0848 | 0.01266 | 0.4814 | 0.694 | 0.0 |
| B | flat | 6.772 | −3.2135 | −0.47452 | 10.4073 | 3.226 | 0.0 |
| B | exp_decay(0.9) | 6.772 | −2.8128 | −0.41534 | 8.0025 | 2.829 | 0.0 |
| B | exp_decay(0.7) | 6.772 | −1.6698 | −0.24656 | **2.9870** | 1.728 | 0.0 |

### MSE heat table (2 × 3)

| | flat | exp_decay(0.9) | exp_decay(0.7) |
|---|---:|---:|---:|
| **DGP-A (stable)** | **0.252** | 0.275 | 0.481 |
| **DGP-B (break)** | 10.407 | 8.003 | **2.987** |

### Acceptance verdict

| Criterion | Threshold | Observed | Pass |
|---|---|---|:---:|
| Under DGP-A, flat ≤ exp_decay(0.7) | decay wastes info on stable | 0.252 ≤ 0.481 | ✅ |
| Under DGP-B, exp_decay(0.7) ≤ flat | decay helps under break | **2.987 ≤ 10.407** | ✅ |
| Under DGP-B, monotone ordering flat ≥ exp9 ≥ exp7 | intermediate | 10.407 ≥ 8.003 ≥ 2.987 | ✅ |
| Under DGP-A, abs(rel_bias) ≤ 0.15 for all weights | bias stable | max = 0.013 | ✅ |

**All 4 Q4 criteria PASS under the corrected DGP-B.** (Prior run: 2/4, with DGP-B criteria
untestable due to mean-shift design flaw.)

### Interpretation

Under DGP-B (σ-shift break), the flat-weight estimator is severely biased downward: it
averages across 6 pre-break years (θ ≈ 2.67) and 4 post-break years (θ ≈ 6.77), producing
a weighted mix well below the target post-break value of 6.77. The bias of −3.21 represents
a 47% underestimate. Exponential decay progressively concentrates weight on the post-break
years, reducing both the absolute bias (−3.21 → −1.67 for exp7) and MSE (10.41 → 2.99).
The improvement is dramatic: exp_decay(0.7) achieves a 71% MSE reduction vs flat under the
σ-shift break, confirming the theoretical motivation for the decay default.

Under DGP-A (stable), the decay trade-off penalty is moderate: exp_decay(0.7) increases MSE
by 91% (0.481 vs 0.252) by discounting valid older data. The flat estimator is clearly
preferred when there is no structural break. This is the canonical bias-variance trade-off.

**Summary:** Under stable DGP, `flat()` has lowest MSE. Under σ-shift break DGP,
`exp_decay(0.7)` has lowest MSE, with `exp_decay(0.9)` intermediate. The simulation
strongly supports the use of exponential decay weighting when a structural break in spread
is present, and validates the decay feature's design rationale.

---

## 5. Q5 — Expected-Range Anchor Verification

**Verdict: ✅ PASS (1/1)**

Results unchanged from 2026-04-16 run. See prior simulation.md §5 and
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q5_anchors.csv`

Summary: max abs error = 3.71e-04 < 1e-3 threshold across all 6 anchor values.

---

## 6. Q6 — Method Comparison

**Verdict: ✅ PASS (4/4 criteria)**

Results unchanged from 2026-04-16 run. See prior simulation.md §6, plus:
- `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q6_methods.csv`
- `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denominators-2026-04-16/q6_methods_heavy.csv`

Summary: `sd` has lowest relative RMSE under Gaussian DGP; `ols` is marginally more robust
under heavy-tailed t(3) totals. OLS MSE ≤ gap MSE at n_years=15. trimmed_gap MSE ≤ gap MSE.

---

## 7. Q7 — Per-Category Override Sanity (FRESH — corrected DGP-B)

**Setup:** Two-category DGP (HR stable, SB with σ-shift break), n_teams=12, n_years=10,
R=2000. Three scenarios toggle `category_spec` for SB only. Break at year index 7 (year 2006).

- **HR:** stable σ=25, θ_HR = 6.6968 (same as DGP-A)
- **SB:** σ-shift break (pre σ=10, post σ=25), θ_SB = 6.7722 (post-break target)

**Scenarios:**
- **S1:** No override, global flat weights. SB biased toward pre-break θ ≈ 2.67.
- **S2:** SB override: `after(2005)` (post-break years only) + `exp_decay(0.7)`. Uses only the 4 post-break years with decay.
- **S3:** SB override: `exp_decay(0.7)` on all years. Partial downweighting of pre-break years.

**Correction from prior run:** The 2026-04-16 Q7 used a mean-shift SB break (mean 100→140,
σ=20 constant). The mean-shift does not change the OLS denominator target, so overrides
produced no benefit or harm to SB MSE. The corrected σ-shift DGP produces a genuine SB
denominator regime change and valid test of the override criteria.

### Results table

| scenario | category | θ | bias | mse | mse_rel_to_S1 | failure% |
|---|---|---:|---:|---:|---:|---:|
| S1 (no override) | HR | 6.697 | 0.0459 | 0.2579 | 1.000 | 0.0 |
| S1 (no override) | SB | 6.772 | −3.2181 | 10.4376 | 1.000 | 0.0 |
| S2 (SB: after+exp7) | HR | 6.697 | 0.0485 | 0.2603 | 1.009 | 0.0 |
| S2 (SB: after+exp7) | SB | 6.772 | 0.0617 | 0.6733 | **0.065** | 0.0 |
| S3 (SB: exp7 all) | HR | 6.697 | 0.0534 | 0.2571 | 0.997 | 0.0 |
| S3 (SB: exp7 all) | SB | 6.772 | −1.6686 | 2.9799 | **0.286** | 0.0 |

### Acceptance verdict

| Criterion | Threshold | Observed | Pass |
|---|---|---|:---:|
| S2 SB MSE < S1 SB MSE (override improves SB) | strict improvement | 0.673 < 10.438 | ✅ |
| HR MSE unchanged (isolation) | abs change ≤ 5% | (0.260−0.258)/0.258 = 0.9% | ✅ |
| S3 SB MSE intermediate: S1 ≥ S3 ≥ S2 | partial improvement | 10.44 ≥ 2.98 ≥ 0.67 | ✅ |

**All 3 Q7 criteria PASS under the corrected DGP-B.** (Prior run: 1/3, with S2 and S3
improvements untestable due to mean-shift design flaw.)

### Interpretation

**S2 (post-break years only + exp_decay(0.7)):** Restricting SB to the 4 post-break years
with exp_decay(0.7) allows the estimator to track the current σ=25 regime almost precisely.
SB bias collapses from −3.22 to +0.06 (virtually zero), and MSE drops by 93.5%
(10.44 → 0.67). This is the "surgery" scenario: the override precisely targets the regime
change with an appropriate date cutoff.

**S3 (exp_decay(0.7) on all years):** Using exp_decay(0.7) without the year cutoff retains
all 10 years but heavily downweights the 6 pre-break years. SB bias improves from −3.22 to
−1.67, and MSE drops by 71.5% (10.44 → 2.98). This is the "blunt instrument" scenario:
significant improvement over flat, but not as sharp as the targeted override.

**HR isolation:** HR MSE changes by less than 1% across all scenarios (S1: 0.258, S2: 0.260,
S3: 0.257), confirming that `category_spec` overrides are isolated to the target category
and do not contaminate other categories' estimates.

**Summary:**
- ✅ The `after()` year cutoff combined with exp_decay is highly effective at eliminating
  structural-break bias in SB (93.5% MSE reduction vs flat baseline).
- ✅ Decay alone (without year cutoff) provides substantial partial improvement (71.5% MSE
  reduction), offering a useful intermediate option when the break year is uncertain.
- ✅ `category_spec` override isolation is confirmed: ≤1% HR MSE change across all scenarios
  (threshold 5%).

---

## 8. Global Observations (This Run)

### 8.1 Pass / fail summary (full study suite)

| Study | Core criteria | Result (this run) |
|---|---|---|
| Q1 — OLS Convergence | 4/4 | ✅ (from 2026-04-16) |
| Q2 — Bias Direction | 3/3 | ✅ (from 2026-04-16) |
| Q3 — Bootstrap Coverage | 0/3 tight/nominal, borderline floor | ⚠️ (from 2026-04-16) |
| Q4 — Decay Trade-off | **4/4** | ✅ **PASS (corrected DGP-B)** |
| Q5 — Anchors | 1/1 | ✅ (from 2026-04-16) |
| Q6 — Method Comparison | 4/4 | ✅ (from 2026-04-16) |
| Q7 — Override Sanity | **3/3** | ✅ **PASS (corrected DGP-B)** |

### 8.2 OLS estimand scale (repeated for Tester)

The OLS denominator targets ~0.27·σ at n_teams=12, NOT (n-1)·σ/E[R_n]≈3.36·σ. This
10–12× gap is a property of the estimator, not a defect. The Tester's calibration check
should use the relative gate (gap/theta_pre > 50%) rather than the absolute threshold
(gap > 5 units) stated in the mailbox handoff — the absolute threshold was derived from
the SD-method analytical formula and does not apply to OLS.

### 8.3 Failure rates

Zero failures across all 18,000 replications (Q4: 12,000; Q7: 6,000). Consistent with
the 2026-04-16 run which showed 0.000 failure rate in all ≈115k runs.

### 8.4 Recommendation on default weights

The Q4 and Q7 results under the corrected DGP-B strongly validate using exponential decay
when a structural break in spread is known or suspected. The current default
`weights = flat()` is optimal only under stable conditions (DGP-A). Given that real-world
SB categories often exhibit structural spread changes (rule changes, era effects), the
simulation evidence supports `exp_decay(0.9)` as the default — it achieves substantial
MSE reduction under break (8.003 vs 10.407, −23%) with only a small cost under stable
conditions (0.275 vs 0.252, +9%). `exp_decay(0.7)` is more aggressive (71% break
improvement, 91% stable cost) and appropriate when the break year is known or the user
supplies `after()` in `category_spec`. This recommendation is for Reviewer consideration;
no spec.md change in this run.

---

## 9. Code Summary

### Files created in this run

| File | Description |
|---|---|
| `simulate_q4q7.R` | Main simulation script (Q4 + Q7 only; DGP-B corrected) |
| `q4_decay.csv` | Q4 result table (6 rows: 2 DGPs × 3 weight schemes) |
| `q7_override.csv` | Q7 result table (6 rows: 3 scenarios × 2 categories) |
| `simulation_q4q7_results.rds` | Bundled R list: θ calibration + q4_df + q7_df |
| `simulation.md` | This report |

### Files referenced (not modified)

| File | Description |
|---|---|
| `q1_convergence.csv` (prior dir) | Q1 results |
| `q2_bias.csv` (prior dir) | Q2 results |
| `q3_coverage.csv` (prior dir) | Q3 results |
| `q5_anchors.csv` (prior dir) | Q5 anchor verification |
| `q6_methods.csv` (prior dir) | Q6 Gaussian results |
| `q6_methods_heavy.csv` (prior dir) | Q6 heavy-tail supplement |

### DGP changes (vs 2026-04-16)

| Component | 2026-04-16 | This run |
|---|---|---|
| `dgp_break()` | `pre_mean=100, post_mean=140, sigma=20` (mean-shift) | `cat_mean=100, pre_sd=10, post_sd=25` (σ-shift) |
| `dgp_two_cat()` | `sb_mu=100→140, sigma=20 constant` | `sb_sd=10→25, mean=100 constant` |
| Calibration gate | — | Relative gap > 50% (appropriate for OLS scale) |

### Seed compatibility

`scenario_seed()` function and MASTER_SEED (`2026041601`) are identical to 2026-04-16.
Q4 uses `study_id=4`, Q7 uses `study_id=7` — same offsets as prior run. The MASTER_SEEDS
vector is drawn first (before any calibration), preserving the mapping from `(study_id,
scenario_idx, rep_idx)` to seed values.

---

## 10. Smoke Run (embedded)

The main execution IS the smoke run for this re-run study. The script ran in 8.6 minutes
with zero failures. Key reproducibility check: running `simulate_q4q7.R` a second time with
the same seeds produces identical output (deterministic via `set.seed()` before each rep).

---

**Verdict: SIMULATED** — Code written and executed, results clean, acceptance criteria
assessed. Q4: 4/4 pass. Q7: 3/3 pass. DGP-B correction validated.
