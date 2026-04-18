<!-- filename: 2026-04-18-sgp-denom-weights-guidance-docs.md -->

# Log Entry: sgp-denom-weights-guidance-docs-2026-04-17

**Date:** 2026-04-18
**Branch:** `feature/sgp-denom-weights-guidance-docs`
**Base:** `develop`
**Workflow:** 3 (docs-only)
**Agents dispatched:** Scriber (implementer + recorder)

---

## What Changed

Added a "Choosing weights" subsection to the `@details` block of `sgp_denominators()` in
`R/sgp-denominators.R`. Regenerated `man/sgp_denominators.Rd` via `devtools::document()`.
No source code, tests, simulation, NEWS entry, or public interface changes.

---

## Files Changed

| File | Change Type | Description |
|---|---|---|
| `R/sgp-denominators.R` | Edit | Added "Choosing weights" subsection to `@details`; roxygen-only, no code change |
| `man/sgp_denominators.Rd` | Regenerated | Output of `devtools::document()` reflecting the new `@details` content |

---

## Process Record

### Proposal

From `request.md` and `impact.md`: R7 per `plans/sgp-cleanup.md`. Surface the Q4/Q7 findings
from `sgp-denom-dgpb-rerun-2026-04-17` in `?sgp_denominators` `@details` so users know which
`weights` preset fits which regime. Scope: three presets (`flat`, `exp_decay(0.9)`,
`exp_decay(0.7)`), regime guidance, Q4 Monte Carlo citation, and one sentence on the Q7 SB
override pattern with HR direction isolation.

The dispatch resolved a prior scope conflict: the code default is `exp_decay(0.9)` (confirmed
at function signature line), not `flat`. Documentation must label `exp_decay(0.9)` as the
default and not relabel `flat` as such.

### Implementation Notes

- Inserted the new `## Choosing weights` section between the `## Dependency notes` section
  and `@section Caveats:`, preserving existing section ordering.
- Used idiomatic `cal_spec(SB = cal(years = after(2005), weights = exp_decay(0.7)))` syntax
  for the SB override pattern, matching Q7's S2 scenario (break at year index 7 = calendar
  year 2006; `after(2005)` selects post-break years only).
- Cited `runs/sgp-denom-dgpb-rerun-2026-04-17` as the validation reference (not NEWS, per
  hard constraint).
- Quantified the Q4 result (71% MSE reduction under sigma-shift DGP) and the Q7 result
  (93.5% MSE reduction, HR MSE change < 1%) using numbers from `simulation.md` §4 and §7.
- No change to the `weights` default at the function signature.
- No change to `@param weights`.

### Validation Results

This is a docs-only workflow. No tester was dispatched. The following checks were performed
by scriber:

**`devtools::document()` output:**

```
ℹ Updating rotostats documentation
ℹ Loading rotostats
Writing 'sgp_denominators.Rd'
```

Only `man/sgp_denominators.Rd` was written. No other Rd files were touched.

**`git diff --stat`:**

```
R/sgp-denominators.R    | 23 +++++++++++++++++++++++
man/sgp_denominators.Rd | 25 +++++++++++++++++++++++++
2 files changed, 48 insertions(+)
```

Exactly the two files specified in `impact.md`. No other files changed.

**`devtools::check()` result:**

```
0 errors | 0 warnings | 4 notes
```

All 4 NOTEs are pre-existing from `origin/develop`:
- Hidden `.git` directory
- Non-standard top-level files (`ARCHITECTURE.md`, `HANDOFF.md`, `api_files`, `specs`)
- Empty `NEWS.md`
- Escaped `\$` in `replacement_from_prices.Rd` and `replacement_level.Rd`

No new NOTEs introduced by this change.

### Problems Encountered and Resolutions

No problems encountered. No BLOCK, HOLD, or STOP signals were issued. The dispatch pre-resolved
the scope conflict (weights default label) and provided exact syntax guidance for the SB
override pattern.

### Review Summary

Pending — reviewer review follows scriber.

---

## Design Decisions

1. **Section placement**: "Choosing weights" is placed after `## Dependency notes` and before
   `@section Caveats:`. This ordering puts prescriptive guidance (what to do) after descriptive
   content (how things work) and before cautionary notes (what to watch out for). Users reading
   top-to-bottom encounter the guidance in a natural learning sequence.

2. **Quantified validation references**: Rather than vague "validated by simulation," the text
   cites specific numbers (71% MSE reduction, 93.5% MSE reduction, < 1% HR change) from the
   actual simulation run. This lets users assess the strength of evidence without needing to
   read the full run directory.

3. **`after(2005)` not `after(2006)` in the example**: Q7's S2 scenario used `after(2005)` to
   select the 4 post-break years (break at year index 7, calendar year 2006). The `after()`
   helper selects years strictly after its argument. The idiomatic form `after(2005)` is
   therefore correct and matches the actual scenario validated.

4. **No NEWS entry**: Per hard constraint and user choice (R7 scope: non-behavioral,
   non-user-breaking guidance enhancement only).

---

## Handoff Notes

- The "Choosing weights" guidance is now visible at `?sgp_denominators` under `## Choosing weights`
  in the Details section.
- The `after(2005)` year in the example is specific to Q7's simulated break year; users
  applying this pattern to real leagues should substitute the actual break year for their
  league's context.
- The `flat()` vs `exp_decay` trade-off (bias-variance) documented here is theoretically
  motivated and Monte Carlo confirmed; no further simulation is needed to support the guidance.
- `exp_decay(0.9)` remains the function default. If a future run changes the default, the
  "Choosing weights" label "(the default)" will need updating.
