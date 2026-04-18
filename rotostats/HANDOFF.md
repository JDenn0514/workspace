# Handoff Notes

**Run:** sgp-denom-weights-guidance-docs-2026-04-17
**Date:** 2026-04-18
**Slug:** 2026-04-18-sgp-denom-weights-guidance-docs
**Branch:** feature/sgp-denom-weights-guidance-docs @ 100ea94 -> PR #15 against develop
**Verdict:** PASS

---

- The "Choosing weights" guidance is now visible at `?sgp_denominators` under `## Choosing weights`
  in the Details section.
- The `after(2005)` year in the example is specific to Q7's simulated break year; users
  applying this pattern to real leagues should substitute the actual break year for their
  league's context.
- The `flat()` vs `exp_decay` trade-off (bias-variance) documented here is theoretically
  motivated and Monte Carlo confirmed; no further simulation is needed to support the guidance.
- `exp_decay(0.9)` remains the function default. If a future run changes the default, the
  "Choosing weights" label "(the default)" will need updating.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/15
**Commit:** `100ea94` docs(sgp): add Choosing weights guidance to sgp_denominators @details
