<!-- filename: 2026-04-18-replacement-multi-pos-all-spec.md -->
# 2026-04-18 — replacement_level() multi_pos = "all" Design Spec

> Run: `replacement-multi-pos-all-spec-2026-04-17` | Profile: `r-package` | Verdict: Pending — reviewer review follows scriber.

## What Changed

Produced `specs/spec-replacement-multi-pos-all.md` — a complete design doc for the
`multi_pos = "all"` mode of `replacement_level()`. This mode computes per-position
replacement-level comparisons for every eligible position a player holds, returning a
3D (player × position × metric) output instead of the single best-assignment result
produced by `multi_pos = "best"`. The spec covers output shape, zero-sum invariant
generalization, downstream function contracts for `par()` / `zar()` / `dollar_values()`,
and one new error class. No code was written.

## Files Changed

| File | Action | Description |
|------|--------|-------------|
| `specs/spec-replacement-multi-pos-all.md` | Created | Primary design doc — four design questions fully resolved |
| `plans/error-messages.md` | Modified | Added row: `rotostats_error_multi_pos_all_unsupported` |
| `ARCHITECTURE.md` (target repo) | Modified | Run header; "all" mode designed note; new Key Design Decisions section |
| `ARCHITECTURE.md` (run directory) | Created | Copy of target repo version for reviewer verification |

## Process Record

### Proposal

**Request scope** (from `request.md` and `plans/replacement-cleanup.md` §R4):

Draft `specs/spec-replacement-multi-pos-all.md` covering four topics:

1. Output shape: 3D array vs. long-form tidy frame — pick one, justify, document trade-offs,
   cover `position_assignments` attribute changes.
2. Zero-sum invariant generalization: the scalar invariant `abs(sum(slots × premium)) < 1e-6`
   must be extended to handle fractional/multi-counted players under `"all"` mode.
3. Downstream contracts for `par()`, `zar()`, `dollar_values()`.
4. New error/warning classes (if any); register in `plans/error-messages.md`.

This is a docs-only workflow (no builder, no tester, no simulation). The deliverable prepares
a future Planner agent with sufficient detail to implement the mode without further user input.

**Parent spec:** `specs/spec-replacement.md` — the `"all"` enumeration is mentioned in one
sentence ("output shape changes to player × position × stat"). This spec expands that sentence
into a complete design.

**Sibling specs reviewed for style:** `specs/spec-replacement-historical-priors.md` (R2),
`specs/spec-replacement-sgp-pool-rate-method.md` (R3). Both confirmed the house style:
formal algorithm section with numbered steps, Failure Modes table, Assumptions table,
Implementation Steps for a Planner, Known Validity Threats.

### Implementation Notes

**Design question 1 — Output shape:** Chose long-form tidy data frame. Key reasoning:
(a) ineligible `(player, position)` pairs produce no row — no NA fill required;
(b) standard dplyr operations vs. array-subscript syntax that appears nowhere else in the
codebase; (c) ~3-4x smaller memory footprint for typical 15-team leagues where most players
have `n_eligible = 1`. The 3D array was rejected on all three counts.

`position_assignments` attribute changes from a named character vector to a named list of
character vectors (one entry per player, each a `character(n_eligible)` vector). Rationale:
dropping the attribute entirely would produce silent NULL reads for callers that don't check
`params$multi_pos`; the list form preserves the attribute contract while conveying the
multi-position structure correctly.

**Design question 2 — Zero-sum invariant:** Fractional allocation: player `i` contributes
`1/n_eligible[i]` to each eligible position's effective slot count (`f_slots[p]`). The
generalized invariant is `abs(sum(f_slots[p] * scarcity_premium[p])) < 1e-5`. This choice
is unique: it sums to 1.0 per player, is distribution-free, and reduces to the scalar
`"best"` invariant when every player has `n_eligible = 1`. Tolerance loosened from `1e-6`
to `1e-5` to accommodate fractional floating-point accumulation; worst-case accumulation
for 500 players × 3 positions is well under `1e-10`, giving a 100x margin.

**Design question 3 — Downstream contracts:** All three downstream functions
(`par()`, `zar()`, `dollar_values()`) reject `"all"` input with a typed error
(`rotostats_error_multi_pos_all_unsupported`). Two alternatives were evaluated and rejected
for each:
- *Aggregate by max-premium position*: silently changes semantics; callers expecting
  multi-position structure get `"best"` behavior without warning.
- *Return per-position output (tall frame)*: wrong abstraction — `par()` and `zar()` produce
  one value per player for auction pricing; `"all"` mode is diagnostic.

**Design question 4 — New error classes:** One new class,
`rotostats_error_multi_pos_all_unsupported`, shared across all three downstream functions.
Two other candidate classes were evaluated and rejected:
- `rotostats_error_fractional_sum_tolerance_exceeded`: redundant with
  `rotostats_error_zero_sum_violation` (same logical error, just with a loosened tolerance).
- `rotostats_error_attribute_shape_mismatch`: `position_assignments` shape change is a
  design consequence, not an error condition.

### Validation Results

**Per-Test Result Table:**

N/A — docs-only workflow. No tests were run.

**Before/After Comparison Table:**

N/A — new design doc with no baseline.

Additional notes:
- No `devtools::document()` or `devtools::check()` runs required (no R code changed).
- The spec was checked against the parent spec's interface section and all four downstream
  function specs (`spec-par.md`, `spec-zar.md`, `spec-dollar-values.md`, `spec-sgp.md`) for
  consistency.

### Problems Encountered and Resolutions

No problems encountered.

The worktree was initialized on an old commit (`4f814e5`, from the early `sgp-denominators`
run). The feature branch was created from `origin/develop` (`f046062`) rather than the local
worktree-pinned commit, which gave access to all current files including the up-to-date
`plans/error-messages.md`, `ARCHITECTURE.md`, and sibling specs.

### Review Summary

Pending — reviewer review follows scriber.

## Design Decisions

1. **Long-form tidy data frame over 3D array**: Three concrete trade-offs: no NA fill for
   ineligible cells, standard dplyr idioms vs. array syntax, 3-4x smaller memory. A 3D array
   is natural for fully-defined cells (panel data); eligibility is sparse and the primary
   consumer operations are row-filter, not positional slice. Decision is auditable in spec §1.

2. **Fractional allocation (`1/n_eligible`) for zero-sum invariant**: Uniquely satisfies:
   sums to 1.0 per player, distribution-free, reduces to scalar `"best"` invariant. Tolerance
   `1e-5` gives a 100x margin above worst-case floating-point accumulation. This approach is
   consistent with how multi-eligible players are handled in other sports-analytics boundary
   contexts (fractional counting is standard in linear programming roster-optimization).

3. **Reject-not-aggregate for downstream functions**: The cleanest contract boundary. Any
   aggregation inside `par()` is a silent semantic change; returning per-position PAR is the
   wrong abstraction for `par()` (which exists to produce one-per-player valuations for
   auction pricing). Explicit rejection with a typed error forces the user to make an
   intentional choice.

4. **Single shared error class `rotostats_error_multi_pos_all_unsupported`**: Shared across
   `par()`, `zar()`, `dollar_values()` because the trigger condition is identical and recovery
   guidance is the same. Callers can `tryCatch(class = "rotostats_error_multi_pos_all_unsupported")`
   once to handle all three cases. The class name encodes the user-visible concept ("all" mode
   is unsupported by these functions) rather than the internal mechanism.

5. **`multi_pos` recorded in `params` for ALL modes**: Not just `"all"`. This is a
   backward-compatible addition that enables defensive checks in downstream functions
   (`params$multi_pos == "all"` without inspecting `replacement_stats` column structure). The
   field does not currently exist in the parent spec's `params` list — the implementation run
   builder must add it.

## Handoff Notes

**For the implementation run (Planner):**

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

6. **The zero-sum invariant tolerance change** (`1e-6` → `1e-5`) applies only to `"all"` mode.
   The `"best"` / `"highest_par"` / `"primary"` / `"custom"` modes retain `1e-6`. The assertion
   code in `R/replacement_internal.R:assert_zero_sum()` must dispatch on `multi_pos` to select
   the correct tolerance.

7. **`dollar_values()` guard fires rarely**: `par()` and `zar()` already reject `"all"` input;
   `dollar_values()` receives their output, not the raw replacement object. The guard in
   `dollar_values()` is defense-in-depth for callers who bypass `par()` / `zar()` (e.g., by
   constructing a valuation object directly or using a custom `replacement_fn`).
