# Handoff Notes

**Run:** sgp-denom-dgpb-rerun-2026-04-17
**Date:** 2026-04-17
**Slug:** 2026-04-17-sgp-denom-dgpb-rerun
**Branch:** feature/sgp-denom-dgpb-rerun @ 9e711aa -> PR #7 against develop
**Verdict:** SHIP (Q4 4/4 PASS, Q7 3/3 PASS)

---

1. **Q4 "decay helps break" criterion confirmed under corrected DGP-B.** Shifting within-year
   sigma (pre_sd=10, post_sd=25) rather than the league mean is the correct DGP-B design.
   The OLS estimator is mean-invariant; only within-year sigma drives the denominator.
   exp_decay(0.7) achieves 71% MSE reduction vs flat under the sigma-break condition.

2. **Q7 category override isolation confirmed.** The `after(2005)+exp_decay(0.7)` override for
   SB achieves 93.5% MSE reduction while perturbing HR by only 0.94% (threshold: 5%).
   Category isolation is robust.

3. **OLS estimand scale lesson for future sim-specs.** The Planner's absolute calibration gate
   (`|theta_post - theta_pre| > 5`) was derived from the SD-method formula and does not apply
   to OLS. Use relative gates for OLS (`(theta_post/theta_pre - 1) > 0.50`). Both Simulator
   (153.2%) and Tester (150.6%) confirmed well above the threshold independently.

4. **No R/ source changes in this run.** All acceptance criteria carried from the prior run's
   123 tests (0 failures). devtools::check() 0E/0W/4N (pre-existing NOTEs only).

5. **Optional follow-up (non-required):** Simulation evidence supports exp_decay(0.9) as a
   more robust default weight than flat() for real-world leagues where structural breaks in
   spread are plausible. ~23% MSE reduction under break at only ~9% cost under stable conditions.
   Requires a spec.md change and a separate run if the user desires it.

6. **Q3 bootstrap undercoverage (carried from 2026-04-16):** Coverage ~84.7% at n_years <= 6,
   nominal 95%. Documented in `?sgp_denominators @section Caveats`. BCa bootstrap deferred.

7. **Full study suite now complete.** Q1/Q2/Q3/Q5/Q6 passed in the 2026-04-16 run (edfa681).
   Q4/Q7 now pass under the corrected DGP-B. The entire sgp_denominators() Monte Carlo
   validation suite has a SHIP verdict.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/7
**Commit:** 9e711aa (tools/simulation/simulate_q4q7.R — only file added to target repo)
**Simulation artifacts:** q4_decay.csv, q7_override.csv, simulation_q4q7_results.rds (workspace only)
