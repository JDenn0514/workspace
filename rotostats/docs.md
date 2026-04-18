# Documentation Summary — sgp-denom-inverse-categories-param-2026-04-17

**Date:** 2026-04-18
**Profile:** r-package
**Verdict:** PASS

---

## ARCHITECTURE.md decision

The existing `ARCHITECTURE.md` (last updated by the `sgp-input-hardening-2026-04-17` run) references
`INVERSE_CATEGORIES` in two places:

1. The Mermaid module diagram's `SGPDH` node label listed `INVERSE_CATEGORIES` as one of the helper
   module's contents.
2. The Module Reference Table row for `R/sgp-denominators-helpers.R` listed `INVERSE_CATEGORIES` in
   the Purpose column.

Because the constant was deleted in this run, both references were updated. The diagram node label now
lists only `METADATA_COLS` and the remaining helper functions. The reference table row now notes that
the constant was deleted and that the direction-flip set is now the `inverse_categories` argument on
`sgp_denominators()`. The `sgp_denominators()` reference table row was also updated to note the new
parameter and marked "YES" in the "Changed in This Run" column.

Additionally, the `sgp_denominators()` call graph sub-diagram was extended to show the new nodes:
`capture missing() flag`, the `validate + normalize inverse_categories` block, the updated OLS step
(rank-flip conditional on `cat %in% inverse_categories`), and the updated sign-check step. Changed
nodes are highlighted in blue (#1e90ff).

A new "Key Design Decisions (sgp-denom-inverse-categories-param-2026-04-17)" section was appended to
record the five design decisions for this run.

**Summary:** ARCHITECTURE.md was modified (not left untouched) because it contained specific language
about the old hardcoded `INVERSE_CATEGORIES` constant.

---

## Files Modified

| File | Action | Description |
|---|---|---|
| `ARCHITECTURE.md` | Modified | Updated SGPDH node label, module reference table, sgp_denominators() call graph; added design decisions section for this run |
| `NEWS.md` | Modified | Expanded inverse_categories bullet to explain default-vs-explicit behavior |

---

## What Changed (user-visible)

`sgp_denominators()` gains a new argument `inverse_categories`. This argument controls which scoring
categories receive a direction-flipped rank before OLS fitting — the mechanism that allows categories
where lower totals are better (ERA, WHIP, OAVG, BB9) to produce correctly-signed SGP denominators.

Previously the set `c("ERA", "WHIP")` was hard-coded inside the package and could not be changed
without modifying package source. It is now a user-facing parameter.

---

## How to Use the New Parameter

### Basic usage (default — no change required)

```r
# ERA and WHIP are still flipped by default. All existing calls are unaffected.
denoms <- sgp_denominators(league_history, scoring_categories = c("HR", "R", "RBI", "ERA", "WHIP"))
```

### Custom inverse categories (OAVG as a lower-is-better stat)

```r
# Batting-average-against is lower-is-better: declare it explicitly.
denoms <- sgp_denominators(
  league_history,
  scoring_categories = c("HR", "R", "RBI", "OAVG"),
  inverse_categories = c("OAVG")
)
```

### Before / After comparison

**Before this change** — adding OAVG required modifying package source:

```r
# No way to score OAVG correctly without editing R/sgp-denominators-helpers.R:
# INVERSE_CATEGORIES <- c("ERA", "WHIP", "OAVG")  # <-- had to edit the package
denoms <- sgp_denominators(league_history, scoring_categories = c("HR", "OAVG"))
# Result: OAVG received no rank-flip → positive OLS slope → sign warning fired.
```

**After this change** — pass `inverse_categories` in the call:

```r
denoms <- sgp_denominators(
  league_history,
  scoring_categories  = c("HR", "OAVG"),
  inverse_categories  = c("OAVG")
)
# Result: OAVG receives rank-flip → negative OLS slope → no sign warning.
```

---

## Acceptance Criterion Scenarios

Three acceptance criteria were tested (T-NEW-01 through T-NEW-04 in the test suite):

1. **Default call unchanged** (T-NEW-01 / T-58): `sgp_denominators(h, scoring_categories = c("HR", "ERA"))` omitting `inverse_categories` produces an ERA slope of -3.333 and denom of 0.300 (tolerance 0.001) — identical to the pre-patch behavior. All 57 pre-existing tests pass.

2. **Custom inverse category works** (T-NEW-03 / T-60): `sgp_denominators(h, scoring_categories = c("HR", "OAVG"), inverse_categories = c("OAVG"))` on monotone OAVG data produces a slope of -40.0 (negative, correct) and denom of 0.025. `rotostats_warning_unexpected_slope_sign` does not fire. Cross-checked against `lm()` reference (tolerance 1e-5).

3. **Invalid element aborts** (T-NEW-04 / T-61): `sgp_denominators(..., inverse_categories = c("NOT_IN_CATEGORIES"))` aborts with `rotostats_error_invalid_inverse_categories`. Error message names the offending element and the valid scored-category set.

---

## Default-vs-Explicit Distinction

This is the key behavioral nuance users should understand:

| Path | How triggered | Membership check | Behavior when ERA/WHIP not scored |
|---|---|---|---|
| Default | `inverse_categories` omitted from call | Silent intersection with `scoring_categories` | Effective set becomes `character(0)` — no abort, no warning; batting-only leagues work unchanged |
| Explicit | `inverse_categories = c(...)` passed explicitly | Strict: aborts if any element not in scored set | Must only pass names that appear in `scoring_categories` |

**Why this distinction?** The original hard-coded constant `INVERSE_CATEGORIES <- c("ERA", "WHIP")` was
applied via `%in%` at runtime — for a batting-only league, no category ever matched, so the constant
contributed no rank flips. A strict membership validation at argument-validation time would have broken
this implicit filtering. The `missing()` primitive preserves the original behavior for the default
while giving users precise control when they supply the argument explicitly.

---

## Migration Note

No migration required. All existing calls to `sgp_denominators()` that omit `inverse_categories` (the
overwhelming majority) are unaffected. The default value `c("ERA", "WHIP")` is the same as the deleted
constant. Passing the default explicitly — `inverse_categories = c("ERA", "WHIP")` — continues to work
for any league that scores both ERA and WHIP. For batting-only leagues, callers must either omit the
argument (default path, silent intersection) or pass only the relevant subset (explicit path,
`inverse_categories = character(0)`).

---

## Documentation Generation

Builder ran `devtools::document()` after adding the `@param inverse_categories` roxygen block.
`man/sgp_denominators.Rd` was regenerated cleanly with no warnings. The `\item{inverse_categories}`
entry is present in the Rd output. Scriber did not re-run `devtools::document()` (no additional
roxygen changes in this dispatch).

---

## Reference

- `ARCHITECTURE.md` — updated in both the target repo root and the run directory (copy for reviewer)
- `log-entry.md` — complete process record with audit trail
- `audit.md` — per-test result table with all tolerances matching test-spec.md verbatim
