# Documentation Changes — replacement-higher-order-cycle-2026-04-17

**Date:** 2026-04-20
**Profile:** r-package
**Verdict:** PASS

## Summary

This run introduces one new user-visible parameter (`cycle_history_window`) and changes
the convergence behavior of `replacement_level()` for `multi_pos = "highest_par"`. All
user-facing documentation has been updated to reflect these changes.

## Files Modified

| File | Action | Description |
| --- | --- | --- |
| `R/replacement.R` — roxygen `@param replacement_params` | modified | Added `cycle_history_window` description (integer, default 5L, range [2, 50], usage guidance) after existing `convergence_max_iter` mention |
| `R/replacement_params.R` — `@format` | modified | Changed "nine elements" to "ten elements"; appended `cycle_history_window` entry description |
| `man/replacement_level.Rd` | regenerated | `devtools::document()` — picks up `cycle_history_window` in `@param replacement_params` |
| `man/default_replacement_params.Rd` | regenerated | `devtools::document()` — picks up updated `@format` with 10 elements |
| `NEWS.md` | verified correct | Builder's "Improvements" bullet is accurate and complete (see below) |
| `ARCHITECTURE.md` | updated | Full update: replaced 2-lag language with state-hash ring buffer description; updated module reference table, function call graph, data flow, key invariants, design decisions |

## Roxygen Documentation Verification

### `replacement_level()` — `@param replacement_params`

Builder added the following text (verified in `R/replacement.R` around line 94):

> `cycle_history_window` (integer, default 5L): depth of the rolling assignment-hash
> history window used to detect higher-order convergence cycles (periods 2 through N)
> in the `multi_pos = "highest_par"` loop. Increase if you observe non-convergence in
> pools with many near-boundary multi-eligible players.

This description is accurate and consistent with the spec. No changes needed.

### `default_replacement_params` — `@format`

Builder updated to "ten elements" and appended:

> `cycle_history_window` (integer, default 5L, rolling assignment-hash history window
> depth for higher-order cycle detection).

This is accurate and consistent with the implementation. No changes needed.

### `man/` files

Both `man/replacement_level.Rd` and `man/default_replacement_params.Rd` were regenerated
by builder via `devtools::document()`. Tester confirmed `devtools::document()` exits clean
with no errors. No manual edits to `.Rd` files required.

## NEWS.md Verification

Builder added the following under "## Improvements" in the unreleased section:

```
* `replacement_level()`: The multi-position convergence loop now detects
  higher-order assignment cycles (periods 3–5) using a rolling state-hash
  buffer of depth `replacement_params$cycle_history_window` (default 5).
  Previously only 2-cycles were detected. This eliminates the
  `rotostats_warning_convergence_not_reached` false-alarm that occurred in
  ~3% of highly multi-eligible pools.
```

Verified: this entry is accurate. The claim "~3%" matches the Study C baseline of
`convergence_rate = 0.966` (3.4% non-convergence). The period range "periods 3–5"
(not 2, because 2-cycles were already detected) is precise. The wording "false-alarm"
is accurate — the warning was emitted for convergent pools that were cycling, not
for genuinely non-convergent cases.

No additional NEWS.md entries needed.

## ARCHITECTURE.md Changes

Full update produced. Key changes from prior version (`replacement-multi-pos-all-spec-2026-04-18`):

1. **Header**: updated Run, Branch, Date.
2. **Module Structure diagram**: `RL_BODY` node label updated to include "state-hash cycle detector"; `RL_PARAMS` node updated to "(10 entries incl. cycle_history_window)".
3. **Function Call Graph**: replaced the prior `replacement_level()` call graph (which described `MPOS` as "highest_par + 2-cycle detect") with a full breakdown of H1/H2/H3/H4 blocks showing the ring buffer flow.
4. **Data Flow diagram**: new diagram traces `cycle_history_window` validation → ring buffer init → hash computation → cycle check → push path.
5. **Module Reference Table**: `replacement_level()`, `default_replacement_params`, `dgp_c.R`, and `test-replacement.R` rows updated with "Changed in This Run = YES" and accurate descriptions.
6. **Replacement Level Section**: "Multi-position iteration loop" sub-section updated to describe state-hash mechanism, order-invariant hash formula, push timing, `highest_par` gate, and `rotostats_error_pool_too_small` preservation rationale.
7. **Known Limitations**: item 1 updated from "Study C near-miss (96.6%)" to "Cycles of period > N still hit max_iter" with guidance for users.
8. **Key Design Decisions**: new section "replacement-higher-order-cycle-2026-04-17" added with 6 decisions.

## Vignettes

No vignettes reference the convergence loop mechanics. No vignette changes needed.

The package does not currently have a pkgdown site configured; no pkgdown changes needed.

## devtools::document() Status

Run by both builder (twice, confirmed clean) and tester (confirmed no drift).
Result: `man/replacement_level.Rd` and `man/default_replacement_params.Rd` regenerated.
No manual `.Rd` edits needed or made.

## Deferred Items

- None for this run. All in-scope documentation surfaces have been updated.
- Future: if `multi_pos = "all"` is implemented (deferred per ARCHITECTURE.md Known Limitations item 4), the convergence loop section will need updating to note that the state-hash detector is gated by `multi_pos == "highest_par"` and does not apply to `"all"` mode.

## ARCHITECTURE.md Locations

- Target repo root: `/Users/jacobdennen/rotostats/ARCHITECTURE.md`
- Run directory mirror: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/replacement-higher-order-cycle-2026-04-17/ARCHITECTURE.md`

Both copies are identical and produced in this run.
