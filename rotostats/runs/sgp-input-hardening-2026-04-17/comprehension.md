# Comprehension Record — sgp-input-hardening-2026-04-17

**Date:** 2026-04-17
**HOLD rounds used:** 0

---

## Input Materials Read

| File | Type | Relevant Content |
|------|------|-----------------|
| `request.md` | Run artifact | Scope, acceptance criteria, hard constraints |
| `impact.md` | Run artifact | Affected files, call sites, risk areas |
| `specs/spec-sgp.md` | Authoritative spec | `sgp()` interface, `pool_baseline` values, edge-case behavior |
| `plans/sgp-cleanup.md §R1` | Plan | Authoritative scope and acceptance for this run |
| `plans/error-messages.md` | Canonical class table | Existing error/warning classes; `rotostats_warning_missing_category_column` description |
| `R/sgp.R` (full, lines 1–626) | Implementation | Current function body, all call sites, roxygen |
| `tests/testthat/test-sgp.R` (lines 470–535) | Tests | Section 6: warning condition tests at lines 488, 517 |
| `tests/testthat/test-sgp-integration.R` (lines 510–632) | Tests | TS-10 at 520, TS-11 at 572, TS-12 at 625 |
| `NEWS.md` | Package news | Entry style: bullet per item, plain prose, backtick function names |
| `ARCHITECTURE.md` (lines 104–150, 430–453) | Architecture | `sgp()` call graph diagram; KDD item 3 dual-use note |

---

## Step 0b — Core Requirement Restatement (from memory)

Two surgical changes to `R/sgp.R`, no algorithmic change:

**Change 1 — `pool_baseline` validation.** After Step 1 (column name normalization) and before Step 2 (`rate_conversion` validation), insert a new validation step: check that `pool_baseline` equals exactly `"projection_pool"`. If it does not, call `cli::cli_abort()` with class `rotostats_error_invalid_pool_baseline` and a message naming the valid value(s). This gates all downstream code that depends on `pool_baseline`. The other documented values (`"per_player"`, `"universal_constants"`) currently reach a downstream `rotostats_error_not_implemented` abort inside the `blended_pool` validation chain — the new top-of-function check makes those paths unreachable and becomes the single authoritative gate.

**Change 2 — Warning class split.** The current `rotostats_warning_missing_category_column` is used for two logically distinct conditions:
- (a) A scored category column is entirely absent from `projections` (Step 8, line 342).
- (b) A player has 0 or NA projected IP/AB for a scored rate stat (Steps 14a, 14c, 14d, lines 539, 572, 603).

After the split:
- Condition (a) retains class `rotostats_warning_missing_category_column` — no change to line 342.
- Condition (b) gets the new class `rotostats_warning_zero_playing_time` — lines 539, 572, 603 change class only; messages stay intact.

Both `plans/error-messages.md` and `ARCHITECTURE.md` must be updated to reflect the split.

---

## Step 0c — Comprehension Self-Test

**Q1: Can I restate the core requirement in one paragraph without looking at the source?**

Yes, stated above.

**Q2: Can I write every formula from memory?**

There are no new mathematical formulas in this change. The SGP math (blended-pool rate stat formulas, counting-stat division) is unchanged. The change is purely to error/warning class dispatch logic.

**Q3: Any symbols, terms, or concepts I cannot precisely define?**

None. All relevant terms are fully defined in the source material:
- `pool_baseline`: character scalar controlling pool construction; valid value is `"projection_pool"` (the only implemented one); `"per_player"` and `"universal_constants"` are documented but stub-not-implemented.
- `rotostats_error_invalid_pool_baseline`: new error class for invalid `pool_baseline` values.
- `rotostats_warning_missing_category_column`: existing warning class, narrowed to condition (a) only.
- `rotostats_warning_zero_playing_time`: new warning class for condition (b).

**Q4: Any steps where I would need to make a judgment call not in the source?**

One resolved judgment call — documented explicitly:

**Resolution of downstream `rotostats_error_not_implemented` conflict.** The dispatch prompt hard-rules state: "Every other literal listed in spec-sgp.md's `pool_baseline` documentation (`"per_player"`, `"universal_constants"`) must ALSO abort under the new class, even though they already abort downstream under `rotostats_error_not_implemented`. Rationale: the top-of-function check is the authoritative gate; downstream abort becomes unreachable." This is authoritative — no judgment needed.

**Q5: Can I explain why the change works?**

The `pool_baseline` check is placed after Step 1 (column normalization, which is harmless) and before Step 2 (`rate_conversion` validation). This ensures the check fires regardless of the `rate_conversion` value — even if `rate_conversion` is invalid, an invalid `pool_baseline` is caught first. Wait: the dispatch prompt says "before Step 2 (rate_conversion validation)". On reconsideration: placing `pool_baseline` validation between Step 1 and Step 2 means it fires first. The dispatch prompt confirms: "Bundle the `pool_baseline` check before Step 2 (rate_conversion validation), so invalid `pool_baseline` values are caught regardless of `rate_conversion`."

The warning split is justified because condition (a) and (b) are logically different failure modes: (a) is a data-completeness problem (column absent), (b) is a data-quality problem (playing time missing). They warrant separate `withCallingHandlers` targets.

**Q6: Implicit assumptions in the source material?**

- `pool_baseline` is only consumed by the `blended_pool` code path. The check fires regardless — even when `rate_conversion = "fixed_baseline"` (which ignores `pool_baseline`). This is intentional per the dispatch prompt's rationale: the top-of-function check is the authoritative gate.
- The `"per_player"` and `"universal_constants"` values for `pool_baseline` are documented in spec-sgp.md but not implemented. They would reach a `rotostats_error_not_implemented` at Step 6b if `pool_baseline` validation did not fire first.

---

## Comprehension Verdict

**FULLY UNDERSTOOD.**

All call sites identified:
- Line 342: Step 8 — keep `rotostats_warning_missing_category_column`.
- Line 539: Step 14a ERA zero-IP — change to `rotostats_warning_zero_playing_time`.
- Line 572: Step 14c WHIP zero-IP — change to `rotostats_warning_zero_playing_time`.
- Line 603: Step 14d AVG zero-AB — change to `rotostats_warning_zero_playing_time`.

Insertion point for new validation:
- After line 219 (end of Step 1) and before line 222 (start of Step 2).

The `@section Warnings` roxygen block (lines 137–148) conflates both conditions under one class — it must be rewritten to document two separate bullets under two separate classes.

The `@param pool_baseline` (lines 107–109) documents only valid values, not validation behavior — it must note that any value other than `"projection_pool"` aborts.

The `@details Validation order` block (lines 58–70) lists 5 steps — a new item 1 (or 1a) must be added noting `pool_baseline` is validated before `rate_conversion`.

`ARCHITECTURE.md` line 448 (KDD item 3) explicitly describes the dual-use as a feature — this prose block must be replaced to describe the split. The diagram at line 127 (`F --> F1["cli_warn\nrotostats_warning_missing_category_column"]`) must be updated to show two branches for the two warning classes.

`plans/error-messages.md` requires three surgical edits:
1. Errors table: add `rotostats_error_invalid_pool_baseline` row.
2. Warnings table: narrow the existing `rotostats_warning_missing_category_column` row to column-absent only.
3. Warnings table: add new `rotostats_warning_zero_playing_time` row.

`NEWS.md`: add bullet(s) under the existing `rotostats (development version)` section, following the established style.

Tests:
- `test-sgp.R:488` — zero IP test currently expects `rotostats_warning_missing_category_column`; must change to `rotostats_warning_zero_playing_time`.
- `test-sgp.R:517–527` — zero AB test currently expects `rotostats_warning_missing_category_column`; must change to `rotostats_warning_zero_playing_time`.
- `test-sgp-integration.R:520` — TS-10 at line 520 expects `rotostats_warning_missing_category_column` for SB column absent — this is condition (a); retain class.
- `test-sgp-integration.R:572` — TS-11 at line 572 expects `rotostats_warning_missing_category_column` for zero-IP — this is condition (b); change to `rotostats_warning_zero_playing_time`.
- `test-sgp-integration.R:625` — TS-12 at line 625 expects `rotostats_warning_missing_category_column` for zero-AB — this is condition (b); change to `rotostats_warning_zero_playing_time`.

Three new tests must be added:
- (a) `pool_baseline = "foo"` aborts with `rotostats_error_invalid_pool_baseline`.
- (b) Missing category column fires `rotostats_warning_missing_category_column` AND NOT `rotostats_warning_zero_playing_time`.
- (c) IP=0 fires `rotostats_warning_zero_playing_time` AND NOT `rotostats_warning_missing_category_column`.
