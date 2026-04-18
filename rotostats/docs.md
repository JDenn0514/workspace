# Documentation Changes: sgp-denom-weights-guidance-docs-2026-04-17

**Date:** 2026-04-18
**Workflow:** 3 (docs-only)

---

## Files Modified

| File | Change | Notes |
|---|---|---|
| `R/sgp-denominators.R` | Added `## Choosing weights` subsection to `@details` | Roxygen-only; no code change |
| `man/sgp_denominators.Rd` | Regenerated | Reflects new `@details` content |

---

## Summary of Changes

### `R/sgp-denominators.R` — `sgp_denominators()` `@details`

A new `## Choosing weights` subsection was appended to the `@details` block, placed
between `## Dependency notes` and `@section Caveats:`.

The subsection:

1. Names each of the three standard weight presets with its intended regime:
   - `flat()`: i.i.d. assumption; lowest MSE when no structural break is present.
   - `exp_decay(0.9)`: mild decay; the default; reasonable compromise when a break is
     suspected but not confirmed.
   - `exp_decay(0.7)`: strong decay; use when a within-year dispersion break is known
     or expected; cites Q4 Monte Carlo validation (71% MSE reduction vs flat under
     sigma-shift DGP from `runs/sgp-denom-dgpb-rerun-2026-04-17`).

2. Includes one sentence on the per-category SB override pattern, citing Q7 results
   (93.5% SB MSE reduction, < 1% HR MSE change):
   ```
   category_spec = cal_spec(SB = cal(years = after(2005), weights = exp_decay(0.7)))
   ```

---

## Doc Generation

Run `devtools::document()` from the package root to regenerate Rd files. This was done
as part of the workflow; `man/sgp_denominators.Rd` is current.

No other doc generation commands are required.

---

## Deferred Items

None.

---

## Architecture Diagram

`ARCHITECTURE.md` updated in target repo root to reflect this run.
Run directory copy: `/Users/jacobdennen/.claude/plugins/data/statsclaw-statsclaw/workspace/rotostats/runs/sgp-denom-weights-guidance-docs-2026-04-17/ARCHITECTURE.md`
(See shipper sync step.)
