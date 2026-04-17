# Handoff Notes

**Run:** replacement-2026-04-16
**Date:** 2026-04-17
**Slug:** 2026-04-17-replacement-level
**Branch:** feature/replacement-level @ 21270fb
**Verdict:** PASS_WITH_FOLLOWUP

---

1. **Study C follow-up ticket (recommended):** Higher-order cycle detection in the
   `highest_par` loop. Current 2-lag (`old_old_assignments`) handles the common case
   but misses 3-cycles+ in stress pools. Approach: hash the position assignment state
   (e.g., `digest::digest(new_assignments)`) and check against a small history window.
   Expected to push convergence_rate from 96.6% → 99%+.

2. **Study E follow-up ticket (recommended):** Either tighten DGP-E further (focal
   pitcher further below pool mean to minimize pool composition variance effects) or
   review the acceptance thresholds (median ≤ 5 and pct_within_2 ≥ 0.80 may be more
   appropriate for a static focal pitcher). The algorithm itself is correct (Study D
   confirms boundary indexing).

3. **Deferred features:** Three features have validation guards but incomplete
   implementations: `seed_method = "historical_priors"`, `boundary_rate_method = "sgp_pool"`,
   `multi_pos = "all"`. Each needs a sub-spec before implementation. Do not silently
   remove the validation guards.

4. **`sort_by = "sgp"` with `blended_pool` denominators requires `league_history`:**
   When `replacement_level()` calls `sgp()` internally on pass 2+, it forwards
   `league_history` (if supplied). If the user passes `sgp_denominators` calibrated
   with `rate_conversion = "blended_pool"` but no `league_history`, `sgp()` will abort.
   This constraint is documented in the roxygen `@param sgp_denominators` description;
   downstream `dollar_values()` should always forward `league_history`.

5. **Pipeline-isolation crossover `dgp_e.R`:** Builder touched simulator-owned code
   in `21270fb`. If the simulator is respawned for a future sim run, ensure DGP-E's
   `.FOCAL_PITCHER` constants (ERA=4.70, WHIP=1.40, IP=165, W=8, K=130) and the
   calibration comment are preserved.

6. **`devtools::document()` required after any roxygen2 change** in
   `R/replacement.R`, `R/replacement_internal.R`, or `R/replacement_params.R`.
   The current run produced clean doc generation.
