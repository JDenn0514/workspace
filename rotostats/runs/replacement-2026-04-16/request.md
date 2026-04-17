# Request

Request ID: replacement-2026-04-16
Created: 2026-04-16
Requester: user (tennisbeast14@gmail.com / JDenn0514)
Target Repo: JDenn0514/rotostats (local: /Users/jacobdennen/rotostats)
Workspace Repo: JDenn0514/workspace
Target Branch: feature/replacement-level (cut from develop)
PR Target: develop
Workflow: 11 — Simulation + Code (new estimator + Monte Carlo evaluation)
Profile: r-package

## Summary

Build the `replacement_level()` function for the `rotostats` R package — a per-position
replacement-level stat-line estimator that serves as the zero-dollar baseline for PAR
(Points Above Replacement) in rotisserie auction valuation. Also build the separate
`replacement_from_prices()` function and the internal helpers in `R/replacement_internal.R`.

## Authoritative Inputs

- `/Users/jacobdennen/rotostats/specs/spec-replacement.md` — conceptual spec (AUTHORITATIVE).
  Planner MUST read this in full before any specs are produced.
- `/Users/jacobdennen/rotostats/plans/error-messages.md` — canonical error/warning class registry.
  Scriber MUST add all new classes listed below to this registry.

## Acceptance Criteria

1. **Boundary band with dynamic K cap produces correct replacement stat lines.**
   The roster boundary at each position is the player at rank `n_teams × roster_slots[pos]`
   in the sorted pool. Replacement stat line = mean over a symmetric band of `2K+1` players
   around the boundary, with `K_eff = min(K, floor(n_rostered_pos / 4))`. Counting stats use
   simple means; ERA/WHIP use IP-weighted means; AVG is `sum(H) / sum(AB)` across band players
   (not the mean of AVG values). Cliff detection truncates the band at `j-1` below the
   boundary and applies only to the lower half, only when ≥ `cliff_min_n` players available.

2. **Zero-sum positional-adjustment assertion holds to 1e-6; hitters and pitchers computed
   separately.** `scarcity_premium[pos] = global_replacement - replacement[pos]`. Enforce
   `abs(sum(roster_slots[zero_sum_positions] × scarcity_premium[zero_sum_positions])) < 1e-6`
   where `zero_sum_positions = setdiff(primary_hitter_slots, "C")` when
   `catcher_adjustment_method = "split_pool"` (default) else `primary_hitter_slots`. Primary
   hitter slots = {C, 1B, 2B, 3B, SS, OF, DH} (DH excluded for NL). UTIL/MI/CI are combo
   slots with no independent pool. Hitter/pitcher pools computed separately. Violation →
   `rotostats_error_zero_sum_violation` via `cli_abort()`.

3. **SP/RP always separated with correct role inference and swingman flagging.** If
   `projections` has a `role` column, use directly; else classify by `IP >= sp_ip_threshold`
   (default 100). Pitchers with `80 <= IP <= 120` flagged in `cliff_metric$swingman`.
   Returned `replacement_stats` always includes `IP` for pitcher rows (even if IP is not a
   scored category).

## Hard Constraints

- All parameters explicit per spec's Interface section; no hardcoded defaults beyond the
  spec's named defaults, no global state.
- No row-wise loops — all per-player computation vectorized.
- I/O lives in separate entry-point functions, not inside core function. `replacement_level()`
  is stateless and deterministic given fixed inputs; `dollar_values()` owns the iteration loop.
- Return a named list matching the Outputs section exactly: `replacement_stats`,
  `positional_adjustments`, `cliff_metric`, `two_way_players`, `pool_diagnostics`, `method`,
  `params`. Attributes: `stat_units`, `config`, `projections`, `position_assignments`,
  `converged`, `iterations`, `delta`.
- Errors/warnings via `cli_abort()` / `cli_warn()` with `class=` from plans/error-messages.md.
  Scriber MUST add the following new classes:
    - `rotostats_error_zero_sum_violation`
    - `rotostats_error_rate_method_mismatch`
    - `rotostats_error_missing_sgp_denominators`
    - `rotostats_error_missing_column`
    - `rotostats_error_wrong_column_type`
    - `rotostats_error_unknown_rate_stat`
    - `rotostats_error_pool_too_small`
    - `rotostats_error_kde_no_trough`
    - `rotostats_error_missing_league_history`
    - `rotostats_error_missing_pos_weight`
    - `rotostats_warning_convergence_not_reached`
    - `rotostats_warning_calibration_suppressed`
    - `rotostats_warning_team_total_divergence`
    - `rotostats_warning_name_match_failure`
- Use `checkmate` for input validation; use `rlang::caller_env()` so errors point to user's frame.
- Implement `replacement_from_prices()` as a separate exported function. Shared helpers
  (`format_replacement_output()`, `compute_positional_adjustments()`,
  `assert_replacement_output_contract()`) live in `R/replacement_internal.R`.
- Export `default_replacement_params` and `rate_stat_denominators()`.

## Simulation Scope (for sim-spec.md)

Monte Carlo validation is part of the acceptance bar. Simulator MUST cover:
- Boundary-band stability under projection perturbation
- Zero-sum assertion holds under all `catcher_adjustment_method` values
  (`split_pool`, `positional_default`, `partial_offset`, `none`)
- Convergence of the `multi_pos = "highest_par"` reassignment loop on synthetic
  multi-eligible pools
- Dynamic K cap behavior in thin AL-only pools (12-team AL SS and C)

## Workspace Repo Status

JDenn0514/workspace exists on GitHub. Plugin mode uses
`/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/` as local workspace path.

## Credentials

See credentials.md — target repo PASS (gh CLI, JDenn0514 active), workspace repo reachable.

## Branching

- Feature branch: `feature/replacement-level` cut from `develop`
- PR target: `develop` (NEVER `main` — main is release-only per CLAUDE.md)
