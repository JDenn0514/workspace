# Implementation — sgp-input-hardening-2026-04-17

**Builder:** Claude Sonnet 4.6  
**Date:** 2026-04-17  
**Commit:** `a1a56e1` on `feature/sgp-input-hardening`

---

## Files Modified

### `R/sgp.R`

All changes are surgical; no SGP math was touched and no argument list changes were made.

**1. New Step 1b validation block (inserted between old lines 219 and 222)**

Inserted 11 lines after `names(projections) <- toupper(names(projections))` and before `Step 2 — Validate rate_conversion`:

```r
# -------------------------------------------------------------------------
# Step 1b — Validate pool_baseline
# -------------------------------------------------------------------------
valid_pool_baselines <- "projection_pool"
if (!pool_baseline %in% valid_pool_baselines) {
  cli::cli_abort(
    "{.arg pool_baseline} must be {.val projection_pool}, not {.val {pool_baseline}}. \\
     Other documented values ({.val per_player}, {.val universal_constants}) are not \\
     yet implemented.",
    class = "rotostats_error_invalid_pool_baseline"
  )
}
```

This fires before all other validation, regardless of `rate_conversion`, making `pool_baseline` an unambiguous top-level gate. The downstream Step 6b check (`rotostats_error_missing_config_field`) that previously caught `pool_baseline = "per_player"` or `"universal_constants"` is now unreachable for those invalid values.

**2. Call-site class changes — three sites changed, one NOT changed**

| Original line | New line (post-insertion) | Step | Condition | Class before | Class after |
|---|---|---|---|---|---|
| 342 | 373 | Step 8 | Category column absent | `rotostats_warning_missing_category_column` | **UNCHANGED** |
| 539 | 570 | Step 14a | ERA: player has 0/NA IP | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |
| 572 | 603 | Step 14c | WHIP: player has 0/NA IP | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |
| 603 | 634 | Step 14d | AVG: player has 0/NA AB | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |

Only `class =` was changed at the three modified sites; message text, logic, and surrounding code are untouched.

**3. Roxygen updates**

- `@param pool_baseline` (lines 111–116): replaced single-sentence description with 5-line description noting that only `"projection_pool"` is implemented and any other value aborts with `rotostats_error_invalid_pool_baseline`.
- `@section Warnings:` (lines 138–165): replaced the 2-item `\enumerate{}` block with a `\subsection{}` block for each class (`rotostats_warning_missing_category_column` and `rotostats_warning_zero_playing_time`), matching spec.md §2c exactly.
- `@details` Validation order (lines 60–71): renumbered the 5-item list to 6 items, inserting `pool_baseline` check as item 1 and renumbering the existing `rate_conversion` items 1–5 to 2–6. Changed backtick inline code to `\code{}` style for consistency in the new items.

**Note on `man/sgp.Rd`:** NOT touched. Scriber owns `devtools::document()` regeneration. The roxygen source changes are complete and ready for scriber to rebuild the `.Rd` file.

---

### `plans/error-messages.md`

Three surgical edits:

1. **New error row** appended after `rotostats_error_not_implemented`:
   `rotostats_error_invalid_pool_baseline` | `sgp()` | pool_baseline is not one of the implemented values | Pass `pool_baseline = "projection_pool"` (the default)

2. **Narrowed warning row** for `rotostats_warning_missing_category_column`: removed the "OR a player has 0 or NA projected IP/AB" clause and the "dual use" explanation. Now covers only the column-absent case.

3. **New warning row** for `rotostats_warning_zero_playing_time` inserted immediately after the narrowed missing-column row: covers the 0/NA IP (ERA/WHIP) or 0/NA AB (AVG) case with recovery guidance to use `withCallingHandlers` for suppression.

No other rows were modified or reordered.

---

### `ARCHITECTURE.md`

Two surgical edits to the `## Key Design Decisions (sgp-2026-04-16)` section and the sgp() call graph diagram:

1. **Diagram**: Updated `J --> J5` node label from `"cli_warn zero IP/AB"` to `"cli_warn\nrotostats_warning_zero_playing_time\nzero IP/AB"`. Added a second branch `J --> J5a["cli_warn\nrotostats_warning_zero_playing_time"]` immediately after the `F --> F1` node per spec §4 Edit 1.

2. **KDD item 3**: Replaced the "dual use" paragraph with the split-class description per spec §4 Edit 2.

No other diagram nodes or prose items were modified.

---

### `NEWS.md`

Added a `## Breaking changes` section immediately before `## Implementation notes for maintainers` with two bullets:
- `pool_baseline` top-of-function validation (new abort with `rotostats_error_invalid_pool_baseline`)
- `rotostats_warning_zero_playing_time` split from `rotostats_warning_missing_category_column` with migration guidance for `withCallingHandlers` callers

---

## Unit Tests Written

Per the builder agent definition, builder writes unit tests based on spec.md. However, the dispatch instructions for this run explicitly restrict the test write surface to tester: "builder does not touch test files (that is tester's write surface)" (spec.md §7). Builder therefore does not add tests; tester will add and update tests per test-spec.md.

---

## Known Limitations and Deferred Items

- `man/sgp.Rd` is NOT updated. Scriber must run `devtools::document()` so that the `@param pool_baseline`, `@section Warnings:`, and `@details` changes appear in the built documentation.
- The downstream Step 6b abort paths for `pool_baseline = "per_player"` and `pool_baseline = "universal_constants"` are now unreachable dead code (they would have raised `rotostats_error_missing_config_field` with a misleading message). These paths are not removed here — they are guarded by the new Step 1b gate. If a future maintainer adds support for those `pool_baseline` values, Step 1b's `valid_pool_baselines` vector is the single place to expand.

---

## Design Choices

- **Step numbering "1b"**: Preserves all existing step numbers (Step 2 through Step 15) without renumbering. Inserting as "1b" makes it obvious this is a gate that runs after column normalization but before any method dispatch.
- **`\code{}` in roxygen validation order**: The new items in the `@details Validation order` list use `\code{}` instead of backticks for consistency with the explicit roxygen syntax required in the spec's replacement text.
- **J5a node in diagram**: The spec explicitly called for adding a `J5a` node in addition to updating the existing `J5` node. Both are retained; J5a provides a clean class-name node label at the diagram level for the zero-playing-time warn, while J5 (updated) provides the legacy "zero IP/AB" context label.
