# Simulation Results: replacement_level()

Request ID: replacement-2026-04-16
Date: 2026-04-17 11:10 UTC
R replications per scenario: 500
Master seed: 20260416

---

## Study A: Boundary-Band Stability

Variance of year-over-year delta in replacement stats, K=1 vs K=3.

| Metric | K=1 | K=3 | Ratio (K3/K1) | Threshold | Pass? |
|--------|-----|-----|---------------|-----------|-------|
| var_HR_1B  | 3.25973 | 1.89842 | 0.582383 | < 1.0 | PASS |
| var_ERA_SP | 0.0369426 | 0.0188818 | 0.51111 | < 1.0 | PASS |

Failures — K=1: 336, K=3: 285
Elapsed: 128.2s

---

## Study B: Zero-Sum Invariant

| catcher_method | max_violation | n_violations | Threshold | Pass? |
|----------------|---------------|--------------|-----------|-------|
| split_pool | 3.55271e-15 | 0 | < 1e-6 / == 0 | PASS |
| positional_default | 5.32907e-15 | 0 | < 1e-6 / == 0 | PASS |
| partial_offset | 5.32907e-15 | 0 | < 1e-6 / == 0 | PASS |
| none | 3.55271e-15 | 0 | < 1e-6 / == 0 | PASS |

Elapsed: 122s

---

## Study C: Multi-Eligible Convergence

| Metric | Value | Threshold | Pass? |
|--------|-------|-----------|-------|
| convergence_rate   | 0.966 | >= 0.99 | FAIL |
| median_iterations  |     4 | <= 5    | PASS |
| n_max_iter_hits    | 3 | <= 5    | PASS |

p95_iterations:     6 | Failures: 14
Elapsed: 118.3s

---

## Study D: Dynamic K Cap

| Config | K_eff_expected | K_eff_observed (mean) | pct_correct | Threshold | Pass? |
|--------|----------------|-----------------------|-------------|-----------|-------|
| 12-team AL SS | 3 |     3 |     1 | K==expected, pct==1.0 | PASS |
| 12-team AL C | 3 |     3 |     1 | K==expected, pct==1.0 | PASS |
| 5-team SS | 1 |     1 |     1 | K==expected, pct==1.0 | PASS |

Elapsed: 43.3s

---

## Study E: Rank Invariance

| Metric | Value | Threshold | Pass? |
|--------|-------|-----------|-------|
| median_abs_rank_diff |     3 | <= 2.0 | FAIL |
| pct_within_2         |  0.46 | >= 0.90 | FAIL |

p90_abs_rank_diff:     6 | Failures: 0
Elapsed: 28.1s

---

## Acceptance Criteria Summary

| Study | Metric | Value | Threshold | Pass? |
|-------|--------|-------|-----------|-------|
| A | var_ratio_HR_1B | 0.582383 | < 1.0 | PASS |
| A | var_ratio_ERA_SP | 0.51111 | < 1.0 | PASS |
| B | n_violations[split_pool] |       0 | == 0 | PASS |
| B | n_violations[positional_default] |       0 | == 0 | PASS |
| B | n_violations[partial_offset] |       0 | == 0 | PASS |
| B | n_violations[none] |       0 | == 0 | PASS |
| B | max_zero_sum_violation[all] | 5.32907e-15 | < 1e-6 | PASS |
| C | convergence_rate |   0.966 | >= 0.99 | FAIL |
| C | median_iterations |       4 | <= 5 | PASS |
| C | n_max_iter_hits |       3 | <= 5 | PASS |
| D | K_eff_12_SS (mean) |       3 | == 3.0 | PASS |
| D | K_eff_12_C (mean) |       3 | == 3.0 | PASS |
| D | K_eff_5_SS (mean) |       1 | == 1.0 | PASS |
| D | pct_correct_K_eff |       1 | == 1.0 | PASS |
| E | median_abs_rank_diff |       3 | <= 2.0 | FAIL |
| E | pct_within_2 |    0.46 | >= 0.90 | FAIL |

## Overall: FAIL

13 / 16 criteria passed.

---

## Simulation Design Notes

- DGP: synthetic projections per sim-spec.md §2 (DGP-A through DGP-E)
- Estimator: replacement_level() called as black-box per sim-spec.md §1
- Seed strategy: scenario_seed(study, scenario_idx, r) = 20260416 + study*100000 + scenario_idx*10000 + r
- Error handling: failed replications recorded; excluded from metrics
- Per-replication seed set at DGP call start (not globally)
- No parallelization used (sequential for reproducibility)

