# Handoff Notes

**Run:** replacement-multi-pos-all-spec-2026-04-17
**Date:** 2026-04-18
**Slug:** replacement-multi-pos-all-spec-2026-04-17
**Branch:** feature/replacement-multi-pos-all-spec @ 96ab669 -> PR #17 against develop
**Verdict:** PASS_WITH_NOTE

---

## Design Doc Complete

`specs/spec-replacement-multi-pos-all.md` is the authoritative design for the
`multi_pos = "all"` mode of `replacement_level()`. All four design questions are resolved.
No code was written — a future statsclaw implementation run (Planner → Builder → Tester →
Scriber) will consume this spec.

## For the implementation run (Planner)

1. The spec (`specs/spec-replacement-multi-pos-all.md`) is the authoritative design. Read
   §5 (Implementation Steps) as the starting point for `spec.md` generation.

2. **`multi_pos` in `params`**: Must be added for ALL modes, not just `"all"`. No existing
   code reads `params$multi_pos`, so this is a pure addition.

3. **Fractional pool construction**: The boundary band at position `p` under `"all"` mode
   must count multi-eligible players at `1/n_eligible`. The implementation challenge is
   efficiently maintaining fractional-count sorted pools without duplicating the full player
   set once per position. A single pass through `projections` with a per-position weight
   column is the recommended approach.

4. **`position_assignments` type change**: Any existing code that reads
   `attr(replacement, "position_assignments")` and assumes a named character vector will
   break under `"all"` mode. The guard in `par()` / `zar()` / `dollar_values()` prevents
   this in the primary pipeline, but any custom code (user's own downstream functions)
   will need updating if they use `"all"` output.

5. **Test-spec TS-all-4 is the critical sanity check**: When all players have `n_eligible = 1`,
   `"all"` and `"best"` must produce identical `replacement_stats` (up to the `player_id`
   column addition and row ordering). This confirms the fractional implementation correctly
   degenerates to the scalar case.

6. **The zero-sum invariant tolerance change** (`1e-6` -> `1e-5`) applies only to `"all"` mode.
   The `"best"` / `"highest_par"` / `"primary"` / `"custom"` modes retain `1e-6`. The assertion
   code in `R/replacement_internal.R:assert_zero_sum()` must dispatch on `multi_pos` to select
   the correct tolerance. (Reviewer editorial note: this dispatch requirement is in log-entry.md
   §Handoff Note 6 but is not made explicit in the spec body — Planner must be aware.)

7. **`dollar_values()` guard fires rarely**: `par()` and `zar()` already reject `"all"` input;
   `dollar_values()` receives their output, not the raw replacement object. The guard in
   `dollar_values()` is defense-in-depth for callers who bypass `par()` / `zar()`.

## Reviewer editorial notes (non-blocking, for future spec revision)

1. Spec §5.1 step 4 should add one sentence: "The existing `assert_zero_sum()` helper must
   dispatch on `multi_pos`: use `1e-5` for `"all"` mode, retain `1e-6` for all other modes."
2. `n_band_players` column definition at §1.1 should clarify "(integer count; players with
   fractional allocation each count as 1)" — currently ambiguous.
3. Parent `specs/spec-replacement.md` "player × position × stat" description should be
   reconciled with the tidy-frame decision when that spec is next revised.

## Active follow-ups (carried forward from replacement-dgp-e-calibration)

- **R6 (HIGH):** Higher-order cycle detection in `highest_par` convergence loop.
  Study C convergence_rate = 0.966 (target >= 0.99). Tracked in
  `plans/replacement-cleanup.md`. Not affected by this run.
- **FIXED_POOL_SEED sensitivity (LOW):** rank_diff=2 is at the acceptance boundary.
  Consider choosing a seed that produces rank_diff=1 or 0 for more margin.
- **R4 (COMPLETE):** `multi_pos = "all"` design doc — DONE in this run.
- **R5 (LOW):** `boundary_rate_method = "sgp_pool"` full implementation.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/17
**Commit:** `96ab669` docs(spec): add design doc for replacement_level(multi_pos = "all")
