# Handoff Notes

**Run:** replacement-dgp-e-calibration-2026-04-17
**Date:** 2026-04-18
**Slug:** replacement-dgp-e-calibration-2026-04-17
**Branch:** feature/replacement-dgp-e-calibration @ 0154ca0 -> PR #16 against develop
**Verdict:** PASS_WITH_NOTE

---

## Study E waiver: RESOLVED

The Study E waiver from `replacement-2026-04-16` (pct_within_2=0.46) is resolved.
DGP-E now uses a fixed complement SP pool (FIXED_POOL_SEED=25260416L, N_FIXED_POOL=95).
Results are deterministic across all 500 replications:
- `median_abs_rank_diff` = 2.0 (threshold <= 2) — PASS
- `pct_within_2` = 1.0 (threshold >= 0.90) — PASS

## What changed

- `inst/simulations/dgp/dgp_e.R` — fixed complement SP pool at module level
- `inst/simulations/run_study_e_only.R` — Study-E-only runner (new file)
- No `R/` source code changes

## Active follow-ups (from replacement-2026-04-16 + this run)

- **R6 (HIGH):** Higher-order cycle detection in `highest_par` convergence loop.
  Study C convergence_rate = 0.966 (target >= 0.99). Tracked in
  `plans/replacement-cleanup.md`. Not affected by this run.
- **FIXED_POOL_SEED sensitivity (LOW):** rank_diff=2 is at the acceptance boundary.
  A future run that re-sources `dgp_e.R` should re-run the pre-loop sanity check.
  Consider choosing a seed that produces rank_diff=1 or 0 for more margin.
- **R3 (LOW):** Verify `rotostats_warning_name_match_failure` in source code.
- **R4 (LOW):** `seed_method = "historical_priors"` sub-spec.
- **R5 (LOW):** `boundary_rate_method = "sgp_pool"` full implementation.

## Branch note

The feature branch was cut fresh from develop at `b423053` (after a stale-worktree
repair documented in `shipper-repair.md`). The commit `0154ca0` was pushed to origin
and PR #16 opened against `develop`.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/16
**Commit:** `0154ca0` sim(dgp-e): implement fixed complement SP pool for Study E rank invariance
