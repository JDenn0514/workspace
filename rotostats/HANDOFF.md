# Handoff Notes

**Run:** pvm-2026-04-23
**Date:** 2026-04-24
**Slug:** pvm-2026-04-23
**Branch:** feature/pvm @ 96e57e9 → PR #28 (open, base: develop)
**Verdict:** PASS WITH NOTE

---

## What Was Done

Implemented `pvm()` — the Proportional Value Matrix estimator — as a new function
in `R/pvm.R`. `pvm()` computes per-player, per-category proportional shares of
above-replacement production in budget-fraction units. It consumes a
`replacement_level()` output and returns a data frame with `pvm_[CAT]` columns,
`total_pvm`, and optional `contrib_[CAT]` columns when `include_raw = TRUE`.

**Key artifacts added:**
- `R/pvm.R` (766 lines) — full 11-step algorithm, three `rate_pool` modes, two
  `sub_replacement` modes, `cat_pct` auto/equal/named-vector
- `man/pvm.Rd`, `NAMESPACE` (`export(pvm)`), `NEWS.md`, `ARCHITECTURE.md` updated
- `tests/testthat/test-pvm.R` + `helper-pvm-fixtures.R` — TS-PVM-1..20, PROP-1..6,
  BENCH-1 (130 tests, 130/130 PASS)
- `inst/simulations/sim-pvm.R` + `inst/simulations/dgp/dgp_pvm.R` — 4-study MC harness
- `tests/simulations/sim-pvm-results.rds` + `sim-pvm-summary.csv` — 9,000-draw sweep

Two BLOCK cycles resolved:
1. NA propagation in `total_pvm` matrix multiply (zero-fill before `%*%`) + DGP
   `position_assignments` keyed by player name instead of PLAYER_ID
2. Rate-stat pool formula for `pool_average` and `fixed_baseline`: algebraic
   cancellation collapsed Pool to zero under uniform IP. Fix: `Pool = sum(pmax(ex, 0))`

---

## What Comes Next

PR #28 is open against `develop`. Before merging:

- [ ] Run `devtools::check()` locally — the `R/sgp.R` top-level execution blocker
  is already fixed on `develop` (commit `b946f50`), so check should now pass
- [ ] Update `plans/error-messages.md` "Thrown by" columns for three pvm-relevant
  classes (`rotostats_error_category_mismatch`, `rotostats_error_missing_config_field`,
  `rotostats_error_multi_pos_all_unsupported`) — documentation-only, no code impact
- [ ] OPS test coverage when OPS support ships (forward-compatible path already in
  `R/pvm.R`, no test coverage this run)
