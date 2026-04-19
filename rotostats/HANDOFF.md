# Handoff Notes

**Run:** par-2026-04-18
**Date:** 2026-04-18
**Slug:** par-2026-04-18
**Branch:** feature/par @ 70e13d5 -> PR #18 against develop
**Verdict:** PASS_WITH_NOTE

---

## What Shipped

`par()` is a new exported function in `R/par.R` that computes per-player Points
Above Replacement (PAR) in SGP units. It takes a `replacement_level()` output
and a `sgp_denominators()` output, delegates all SGP conversion to `sgp()`, and
subtracts the position-specific replacement-level SGP from each player's
individual SGP. SP and RP always use separate replacement baselines. The function
includes a band calibration check that warns when the median `total_par` of the
+/-K players around the roster boundary exceeds a configurable threshold.

`rotostats_warning_band_check` was added to `plans/error-messages.md` as a new
warning class entry. A Monte Carlo simulation harness at
`tests/simulations/sim-par.R` (5 scenarios x 200 reps = 1,000 total) validates
the delegation identity invariant, boundary anchoring, and band check behavior.

## For the Next Run (Planner)

- Future `dollar_values()` function will take `par()` output as its primary input and allocate the league's total surplus budget in proportion to each player's `total_par`.
- `par()` rejects `multi_pos = "all"` replacement objects with `rotostats_error_multi_pos_all_unsupported` (design decision from `replacement-multi-pos-all-spec-2026-04-18`). This is not yet tested because the `"all"` mode itself is deferred.
- `replacement_from_prices()` output is intentionally incompatible with `par()` (NULL projections attribute). The error message in `rotostats_error_missing_replacement_attrs` explains this explicitly.
- The band check cannot detect `n_teams` miscalibration — documented in `ARCHITECTURE.md §PAR Section — Known Limitations`. A future `reference_config` parameter is the correct approach.

## Reviewer Notes (deferred, non-blocking)

1. **Summary CSV `ac_sim2_pass` shows FALSE for S1/S3**: Pre-amendment 0.05 blanket tolerance in CSV. Authoritative SV-1 assertions in `test-par-sim.R` use correct position-specific tolerances. Optionally re-run harness post-merge for a clean CSV.
2. **Roxygen gap**: (a) `@details` Step 10 and `@return` describe `na.rm = FALSE` but code uses `na.rm = TRUE`; (b) `@examples` entirely in `\dontrun{}`. Both are pre-CRAN-submission issues. Defer to follow-up scriber commit before `dollar_values()` PR.
3. **HANDOFF.md in target repo root**: Pre-existing since initial scaffolding (2026-04-10). Low-risk project hygiene item.

## Active Follow-ups (carried forward)

- **R6 (HIGH):** Higher-order cycle detection in `highest_par` convergence loop. Study C convergence_rate = 0.966 (target >= 0.99). Tracked in `plans/replacement-cleanup.md`.
- **FIXED_POOL_SEED sensitivity (LOW):** rank_diff=2 is at the acceptance boundary.
- **R5 (LOW):** `boundary_rate_method = "sgp_pool"` full implementation.
- **multi_pos = "all" implementation (SPEC READY):** `specs/spec-replacement-multi-pos-all.md` is the authoritative design. Ready for implementation run.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/18
**Commit:** `70e13d5` docs(par): correct na.rm in total_par roxygen to match implementation
