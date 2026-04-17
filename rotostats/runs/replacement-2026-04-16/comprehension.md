# Comprehension Record

Request ID: replacement-2026-04-16
Agent: planner
Date: 2026-04-17
HOLD rounds used: 0

---

## Input Materials Read

1. `request.md` — run directory. Acceptance criteria, hard constraints, simulation scope, branching instructions.
2. `impact.md` — run directory. Repository survey, affected surfaces, risk areas, pipeline isolation rules.
3. `/Users/jacobdennen/rotostats/specs/spec-replacement.md` — AUTHORITATIVE conceptual spec. Read in full (860+ lines). Covers: formal definition, interface, outputs, multi-pos handling, convergence loop, catcher methods, error table, `replacement_from_prices()`, dependencies, validity threats, validation approach.
4. `/Users/jacobdennen/rotostats/plans/error-messages.md` — canonical error/warning class registry. All 14 new classes to be added by scriber are absent from current registry; confirmed no collisions.
5. `/Users/jacobdennen/rotostats/plans/implementation/league-config-impl.md` — `pool_sizes(config)` spec and `league_config` schema. Confirmed `pool_sizes()` is already implemented at `R/league-config.R:349`; both `sgp()` and `replacement_level()` must call it without duplication.
6. `/Users/jacobdennen/rotostats/plans/implementation/league-history-impl.md` — `league_history` schema. `$team_season` holds team-season totals; `$prices` holds auction history. Used by `replacement_from_prices()` and the $1 calibration check.
7. `/Users/jacobdennen/rotostats/R/league-config.R` — Confirmed `PRIMARY_HITTER_SLOTS`, `CANONICAL_CATEGORIES`, `VALID_PITCHER_SLOT_NAMES`, and internal `pool_sizes()` helper (lines 348–355).
8. `/Users/jacobdennen/rotostats/R/sgp.R` — Confirmed `sgp()` signature. Internal call from iteration loop must pass `projections`, `denominators`, `league_history`, `league_config`. Returns a plain data.frame with `sgp_<CAT>` columns and `total_sgp`.
9. `/Users/jacobdennen/rotostats/R/league-history.R` — Confirmed `$team_season` and `$prices` slot construction and validation.
10. `/Users/jacobdennen/rotostats/CLAUDE.md` — Conventional Commits, `devtools::document()` before commit, `devtools::check()` before PR, `checkmate`, `cli_abort` with `class=`, `rlang::caller_env()`.

---

## Core Requirement Restatement (from memory)

`replacement_level()` is a per-position replacement-level estimator for rotisserie auction leagues. Given a projection data frame and a `league_config`, it identifies the "roster boundary" player at each position (the last player who would realistically be drafted given `n_teams × roster_slots[pos]` depth), averages a symmetric band of `2K+1` players around that boundary into a replacement stat line, applies cliff detection to truncate the lower half of the band if a talent discontinuity is found, computes per-position positional adjustments (scarcity premiums) enforced to zero-sum, separates SP from RP at all times, handles multi-position eligible players via an internal convergence loop (`max_iter = 25`, `tol = 0.01`), and returns a named list with `replacement_stats`, `positional_adjustments`, `cliff_metric`, `two_way_players`, `pool_diagnostics`, `method`, and `params`, plus six attributes. The function manages its own convergence loop internally; callers do not orchestrate iteration.

`replacement_from_prices()` is a separate exported function that derives replacement stat lines from historical `$1` auction prices (trimmed mean) rather than projections. It shares the same output schema via shared internal helpers but has a completely different algorithm and signature.

---

## Key Formulas Verified

### Boundary band

- Boundary index: `b = n_teams * roster_slots[pos]` (when sorted descending by composite rank)
- Band players: positions `b - K_eff` through `b + K_eff` in the sorted pool
- K_eff: `min(K, floor(n_rostered_pos / 4))` where `n_rostered_pos = n_teams * roster_slots[pos]`
- Counting stat replacement: arithmetic mean over band
- ERA/WHIP replacement: IP-weighted mean: `sum(ERA_i * IP_i) / sum(IP_i)` over band
- AVG replacement: `sum(H_i) / sum(AB_i)` over band (not mean of AVG values)

### Cliff detection (lower half only)

- Lower half = band positions `b+1` through `b + K_eff`
- Skip if `|lower half| < cliff_min_n` (default 4)
- MAD method: cliff triggers when gap between adjacent players `>= cliff_threshold * mad(band_values) * 1.4826`; truncate at position `j-1` where j is the cliff position
- Fisher-Jenks: `classInt::classIntervals(band_values, n=2, style="fisher")`; cliff confirmed when split falls at or below boundary player position
- Gap ratio: `cliff_stat = gap_between_adjacent / range(band_values)`; threshold `cliff_gap_ratio_threshold` (default 0.375)

### Positional adjustments (scarcity premiums)

- `scarcity_premium[pos] = global_replacement_level - replacement_level[pos]`
- Global replacement level computed separately for hitters and pitchers
- Zero-sum check: `abs(sum(roster_slots[zero_sum_positions] * scarcity_premium[zero_sum_positions])) < 1e-6`
- `zero_sum_positions = setdiff(primary_hitter_slots, "C")` when `catcher_adjustment_method = "split_pool"` (default), else `primary_hitter_slots`

### Rate stat ranking (boundary_rate_method)

- `raw_ip` (default): `ERA_contribution = (repl_ERA - pitcher_ERA) * pitcher_IP`; `WHIP_contribution = (repl_WHIP - pitcher_WHIP) * pitcher_IP`; `AVG_contribution = (player_AVG - repl_AVG) * player_AB`
- `sgp_pool`: marginal pool-blending (incompatible with `fixed_baseline` denominators)

### Two-way PAR

- `two_way_PAR = hitter_PAR + pitcher_PAR - 1`

### Convergence

- Pass 1: z-score rank; compute initial replacement stats and assignments
- Pass N (N >= 2): call `sgp()` internally; re-rank by `total_sgp`; new boundary; recompute stats; re-evaluate multi-pos assignments
- Converged when BOTH: (1) zero position changes between passes AND (2) `max(abs(new_repl_stats - old_repl_stats)) < tol`

### Pool sizes (from pool_sizes helper)

- `pool_size_p = config$n_teams * sum(config$pitcher_slots)`
- `pool_size_h = config$n_teams * sum(config$roster_slots[intersect(names(config$roster_slots), PRIMARY_HITTER_SLOTS)])`

### Dynamic K cap example (thin pool)

- 12-team AL SS: `n_rostered_pos = 12`, `K_eff = min(3, floor(12/4)) = min(3, 3) = 3`; band of 7 out of 12 = 58%
- 12-team AL C: `n_rostered_pos = 12`, `K_eff = 3`; same band proportion

---

## Comprehension Self-Test Results

1. **Core requirement restated without source:** Yes — done above.

2. **All formulas restated from memory:** Yes — all key formulas above match source precisely. Cross-checked:
   - `K_eff = min(K, floor(n_rostered_pos / 4))`: confirmed (spec §Formal Definition)
   - `AVG = sum(H)/sum(AB)`: confirmed (spec §Formal Definition, request.md §Acceptance Criteria)
   - Zero-sum with `zero_sum_positions`: confirmed (spec §Known Validity Threats, request.md §Acceptance Criteria)
   - IP-weighted ERA/WHIP: confirmed (request.md §Acceptance Criteria)

3. **Undefined symbols:** None. All symbols defined: `K_eff`, `n_rostered_pos`, `scarcity_premium`, `global_replacement_level`, `zero_sum_positions`, `primary_hitter_slots`, `cliff_min_n`, `cliff_threshold`, `tol`, `max_iter`.

4. **Steps requiring judgment calls not in source:**
   - The spec says swingman set must be computed BEFORE role classification (impact.md Risk Area #3). The spec does not spell out the exact ordering of these two steps but the constraint is clear.
   - `replacement_from_prices()` step 4 says "per position (inferred from player eligibility flags in the `prices` data)" — the exact column name for eligibility in `$prices` is not specified. The `league_history$prices` schema (from league-history-impl.md) does not include a `pos_eligibility` column. This is a gap but it is resolvable: the builder can use `pos_eligibility` if present or fall back to `player_type = "batter"/"pitcher"` for high-level SP/RP split. This is a builder-level decision, not a planner blocker — I will document the expectation in spec.md.
   - The spec notes `sort_by = "sgp"` iteration loop: `sgp()` is called with `projections`, `denominators`. The spec does not spell out whether `league_history` and `league_config` are also forwarded to the internal `sgp()` call. From reading `sgp.R`, `sgp()` requires `league_history` and `league_config` when `rate_conversion = "blended_pool"`. Since `replacement_level()` receives `config`, it can forward both. I will specify this in spec.md.

5. **Mathematical/statistical intuition:** Yes. The replacement level concept mirrors WAR: a zero-dollar anchor represents freely replaceable production. The band approach reduces single-player sensitivity. Cliff detection prevents artificially lowering replacement level by including clearly sub-replacement players. Zero-sum enforces that positional adjustments are a redistribution within a fixed hitter/pitcher budget, not a net creation of value.

6. **Implicit assumptions:**
   - `projections` column names are already in the correct case (or will be normalized). The spec says "exact names must match `categories` arg" but does not specify case normalization. From `sgp.R` we see `names(projections) <- toupper(names(projections))` is applied at entry; `replacement_level()` should do the same.
   - `league_history` forwarded to internal `sgp()` calls when `sort_by = "sgp"`. The spec does not explicitly state this but it is required by `sgp()`'s contract. I will call this out in spec.md.
   - `replacement_from_prices()` `prices` input has column `pos_eligibility` or equivalent. Not specified in the schema. I will note this assumption explicitly.

---

## Comprehension Verdict

**UNDERSTOOD WITH ASSUMPTIONS**

All core algorithms are fully specified. Two minor implementation gaps resolved by assumption:

**Assumption A** (`replacement_from_prices()` position eligibility source): The `prices` data frame in `league_history$prices` does not have a documented `pos_eligibility` column. The function must identify each $1 player's position from the price data. I assume the builder will either (a) require `pos_eligibility` in the `prices` data frame or (b) accept a `position` column. Spec.md will state this expectation explicitly, and builder may add it as a required column in `replacement_from_prices()` validation.

**Assumption B** (internal `sgp()` call parameters): When the iteration loop calls `sgp()` internally (`sort_by = "sgp"`), `replacement_level()` forwards `sgp_denominators`, `league_history` (if supplied), and `config` to match `sgp()`'s `blended_pool` contract. The spec does not state this explicitly but it follows from `sgp()`'s requirements. Spec.md will state this explicitly.

**Assumption C** (column name normalization): `replacement_level()` normalizes `projections` column names to uppercase at entry (matching `sgp()` convention), silently. Spec.md will specify this.

No HOLD required. All three assumptions are minor builder decisions that can be specified in spec.md without user input.
