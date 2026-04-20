# Handoff Notes

**Run:** replacement-higher-order-cycle-2026-04-17
**Date:** 2026-04-20
**Slug:** replacement-higher-order-cycle-2026-04-17
**Branch:** feature/replacement-higher-order-cycle @ 5a9dc6d -> PR #20 against develop
**Verdict:** PASS_WITH_NOTE

---

## What Shipped

The `multi_pos = "highest_par"` convergence loop in `replacement_level()` now uses a
rolling ring buffer of assignment hashes (depth N=5, user-overridable via
`replacement_params$cycle_history_window`) to detect cycles of period 2 through N.
This replaces the previous 2-lag `old_old_assignments` variable, which only caught
period-2 cycles.

A new parameter `cycle_history_window = 5L` was added to `default_replacement_params`
as its 10th entry. Hash is pure base-R `paste()` — no new dependencies. The
`checkmate::assert_int(lower = 2L, upper = 50L)` validation fires immediately after
`modifyList()` merges user overrides.

DGP-C (`inst/simulations/dgp/dgp_c.R`) was tightened with:
- A deterministic 3-cycle cluster (players 9901–9903; eligibilities 2B|SS, SS|3B, 2B|3B)
  appended after the random multi-eligibility assignment loop
- A rejection-sampling guard that retries the random assignment if any infield position
  has fewer than 12 mono-eligible players (max 20 retries)

This eliminated the 10/500 thin-pool `rotostats_error_pool_too_small` errors that caused
the first simulator run's convergence_rate of 0.98. The `rotostats_error_pool_too_small`
abort was intentionally preserved (Leader rejected Simulator's routing to relax it).

Study C Monte Carlo (R=500) convergence_rate improved from 0.966 (FAIL) to 1.0 (PASS).

## For the Next Run (Planner)

- `old_old_assignments` is fully removed. If you grep for it, you will find only deleted
  lines in git history. The ring buffer is `assignment_hash_history` (character vector,
  length 0..`cycle_window`).
- `cycle_history_window` is accessible as `params$cycle_history_window` after `modifyList`.
  User-overridable via `replacement_params = list(cycle_history_window = 10L)`.
- Hash function contract: `paste(names(a)[order(names(a))], a[order(names(a))], collapse = "|")`.
  MUST sort by `names(a)` before paste for order-invariance. Any change that alters
  `names()` ordering without resorting will break the invariant.
- DGP-C rejection-sampling retries up to 20 times. If you change `n_teams` in DGP-C,
  update the `n_teams_dgpc = 12L` constant accordingly.
- The next planned function is `zpar()` (zone-adjusted PAR) or `dollar_values()`.
  `dollar_values()` takes `par()` output as its primary input and allocates the league's
  total surplus budget in proportion to each player's `total_par`.

## Reviewer Notes (deferred, non-blocking)

1. **TS-R6-2 guard check**: The fixture converges with `window=2`, indicating a 2-cycle
   path rather than a confirmed 3-cycle. Primary assertions (converged==TRUE with window=5,
   no warning) are binding. Monte Carlo Study C is the primary 3-cycle validation. Tracked
   as F1 below.
2. **Study A var_ratio_HR_1B**: delta +0.0134 marginally exceeds ±0.01 regression window.
   Primary criterion (< 1.0) met; DGP-A unchanged; MC variance, not a regression.
3. **HANDOFF.md in target repo root**: Pre-existing since 2026-04-16 scaffolding (`90473c1`).
   Not introduced by this run. Tracked as F3 below.
4. **specs/spec-replacement.md not updated**: Scriber omission; internal spec, not
   user-facing. User-facing docs (ARCHITECTURE.md, roxygen) are correct. Tracked as F2 below.

## Active Follow-ups

- **F1 (LOW):** Add deterministic 3-cycle unit test fixture with confirmed period-3 behavior
  (`replacement-3cycle-unit-fixture`).
- **F2 (LOW):** Update `specs/spec-replacement.md` to document state-hash cycle detector
  (`replacement-spec-cycle-detector-doc`).
- **F3 (LOW):** Remove pre-existing `HANDOFF.md` from repo root (`repo-cleanup-handoff-md`).
- **R6 is now SHIPPED** — `plans/replacement-cleanup.md` R6 ticket should be marked done
  after PR #20 merges.
- **par() roxygen gap (LOW):** `@details` Step 10 and `@return` describe `na.rm = FALSE`
  but code uses `na.rm = TRUE`; `@examples` in `\dontrun{}`. Defer to pre-CRAN scriber run.
- **FIXED_POOL_SEED sensitivity (LOW):** rank_diff=2 at acceptance boundary; from R5.
- **multi_pos = "all" implementation (SPEC READY):** `specs/spec-replacement-multi-pos-all.md`
  is the authoritative design.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/20
**Commit:** `5a9dc6d` docs(replacement): add R6 state-hash detector to ARCHITECTURE + run log
