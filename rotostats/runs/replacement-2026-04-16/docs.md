# Documentation Changes: replacement-2026-04-16

Request ID: replacement-2026-04-16
Agent: scriber
Date: 2026-04-17

---

## Files Modified

### `plans/error-messages.md` (target repo)

Added 14 new rows to the canonical error/warning class registry:

**Errors (10 new rows):**

| Class | Trigger context |
|---|---|
| `rotostats_error_zero_sum_violation` | Internal assertion after positional adjustment computation |
| `rotostats_error_rate_method_mismatch` | `boundary_rate_method = "sgp_pool"` + `fixed_baseline` denominators |
| `rotostats_error_missing_sgp_denominators` | `sort_by = "sgp"` without `sgp_denominators` |
| `rotostats_error_missing_column` | Required column absent from `projections` or `prices` |
| `rotostats_error_wrong_column_type` | Scored category column is non-numeric |
| `rotostats_error_unknown_rate_stat` | Rate stat not in merged denominator lookup |
| `rotostats_error_pool_too_small` | `n_teams × roster_slots[pos]` exceeds pool size |
| `rotostats_error_kde_no_trough` | `trim_method = "kde"` finds no trough |
| `rotostats_error_missing_league_history` | `seed_method = "historical_priors"` without `league_history` |
| `rotostats_error_missing_pos_weight` | `positional_adjustment_method = "posblend"` without `pos_weight` |

**Warnings (4 new rows):**

| Class | Trigger context | Verbose-gated? |
|---|---|---|
| `rotostats_warning_convergence_not_reached` | Convergence loop hits `max_iter` | No — always fires |
| `rotostats_warning_calibration_suppressed` | `$1` pool below `calibration_min_n` | No — always fires |
| `rotostats_warning_team_total_divergence` | Projected team IP/AB diverges >15% from history | Yes |
| `rotostats_warning_name_match_failure` | Name normalization fails to match a price record | Yes |

All rows follow the existing table format: Class | Thrown by | Condition | Recovery guidance. Rows were appended immediately after the last existing error row (before the `## Warnings` header) and after the last existing warning row respectively.

---

### `ARCHITECTURE.md` (target repo root)

Updated header to reflect `replacement-2026-04-16` run and `feature/replacement-level` branch. Added the following new content:

**Module Structure diagram**: Added `replacement_level()`, `replacement_from_prices()`, `default_replacement_params`, `rate_stat_denominators()` to the API layer subgraph; added `Replacement Core` subgraph containing `RL_BODY`, `RFP_BODY`, `RL_INT`, and `RL_PARAMS` nodes. Added dependency edges: `RL_BODY` → `RL_INT`, `RL_PARAMS`, `pool_sizes()`, `SGP_BODY`. Changed nodes styled in `#1e90ff`.

**Function Call Graph**: Added `replacement_level()` call graph showing the full pipeline from validation through parameter resolution, role inference, convergence loop (band + cliff + stat line + positional adjustments + zero-sum assertion + multi-pos reassignment), convergence check, `format_replacement_output()`, and `assert_replacement_output_contract()`.

**Data Flow diagram**: Added `replacement_level()` branch showing projections + config → z-score/SGP sort → boundary band computation → positional adjustments → convergence loop → named list output.

**Module Reference Table**: Added 8 new rows for the replacement-level surfaces (`R/replacement.R`, `R/replacement_internal.R` mandated helpers + additional helpers, `R/replacement_params.R` both exports). Existing rows updated to "No" for Changed in This Run.

**Replacement Level Section** (new top-level section): Purpose, Input Contract, Algorithm Sketch (boundary band with dynamic K cap, cliff detection, SP/RP role inference, zero-sum positional adjustment, multi-position iteration loop with 2-cycle detection), Output Contract (7 elements + 7 attributes), Key Invariants, Known Limitations and Follow-up Tickets, Cross-References.

**Key Design Decisions (replacement-2026-04-16)**: 7 decisions documented: swingman-before-role ordering, NaN guard in `compute_positional_adjustments()`, `"none"` catcher recentering matching `"split_pool"`, 2-cycle detection via `old_old_assignments`, `projections` attribute stripped before loop, BABIP added to `RATE_STAT_DENOMINATORS`, pipeline-isolation crossover on `dgp_e.R`.

Prior `sgp-2026-04-16` Key Design Decisions moved to their own labeled section for attribution clarity.

---

### `NEWS.md` (target repo)

Added a new entry block at the top of the existing `rotostats (development version)` section (before the `sgp()` bullet), introducing:

- `replacement_level()` — one-sentence description
- `replacement_from_prices()` — one-sentence description
- `default_replacement_params` — one-sentence description
- `rate_stat_denominators()` — one-sentence description

Existing entries for `sgp()` and related functions were not modified.

---

## Documentation Generation

`devtools::document()` was run after all changes. Exit code 0, no errors or warnings. The `plans/error-messages.md` changes do not affect Rd file generation (no roxygen2 tags); `NEWS.md` is a plain Markdown changelog; `ARCHITECTURE.md` is a reference document. No new `.Rd` files were required or generated by scriber's changes.

---

## Architecture Diagram

`ARCHITECTURE.md` produced for both the target repo root and the run directory. See `ARCHITECTURE.md` in this run directory for the run-dir copy.
