# Implementation Spec — sgp-input-hardening-2026-04-17

**For:** Builder  
**Date:** 2026-04-17  
**Workflow:** Code-only (Workflow 2)  
**Branch:** `feature/sgp-input-hardening`

---

## 1. Notation

| Symbol | Meaning |
|--------|---------|
| `pool_baseline` | Character scalar argument to `sgp()`; controls pool construction for blended-pool rate stats |
| `VALID_POOL_BASELINES` | The set `{"projection_pool"}` — the only implemented value |
| `rotostats_error_invalid_pool_baseline` | New error class for invalid `pool_baseline` |
| `rotostats_warning_missing_category_column` | Existing warning class — narrowed to "category column absent from `projections`" |
| `rotostats_warning_zero_playing_time` | New warning class — "player has 0 or NA IP/AB on a scored rate category" |

---

## 2. Changes to R/sgp.R

### 2a. New validation step: `pool_baseline` check

**WHERE:** Insert between the end of Step 1 (line 219, `names(projections) <- toupper(names(projections))`) and the beginning of Step 2 (line 222, `valid_rate_conversions <- c(...)`).

**WHAT to check:** `pool_baseline` must equal `"projection_pool"` exactly.

**Expression:**

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

**Rationale for placement.** This check is placed after column-name normalization (Step 1) and before all other validation. It fires regardless of the value of `rate_conversion` — even if `rate_conversion` is invalid, an invalid `pool_baseline` is caught and reported first. This is intentional: `pool_baseline` is a top-level gate, and catching it early gives the user the most specific error message.

**Effect on downstream stubs.** The `pool_baseline` values `"per_player"` and `"universal_constants"` are documented in `specs/spec-sgp.md` but not implemented. They would previously have reached an abort at the Step 6b block (`rotostats_error_missing_config_field`) or been silently ignored. After this change, those values are caught here under `rotostats_error_invalid_pool_baseline` before reaching any downstream code. The downstream abort paths for those values become unreachable. This is correct — this check is the single authoritative gate.

**Step numbering.** Number this step "1b" in the comment header, preserving the existing step numbers (Step 1 through Step 15) without renumbering.

### 2b. Call-site class changes for zero playing time

Change only the `class =` argument at the following three call sites. **Do not change any message text, logic, or surrounding code.**

| Line | Step | Condition | OLD class | NEW class |
|------|------|-----------|-----------|-----------|
| 539 | 14a | ERA: player has `IP == 0` or `is.na(IP)` | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |
| 572 | 14c | WHIP: player has `IP == 0` or `is.na(IP)` | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |
| 603 | 14d | AVG: player has `AB == 0` or `is.na(AB)` | `rotostats_warning_missing_category_column` | `rotostats_warning_zero_playing_time` |

Line 342 (Step 8, category column absent): **NO CHANGE.** Keep `class = "rotostats_warning_missing_category_column"`.

### 2c. Roxygen updates

**`@param pool_baseline`** (currently at lines 107–109):

Replace the current single-sentence description with:

```
@param pool_baseline Character scalar. Determines how pool constants are
  constructed for the blended-pool rate-stat formulas. Currently only
  \code{"projection_pool"} (the default) is implemented; any other value
  aborts with \code{rotostats_error_invalid_pool_baseline}. The other
  documented values (\code{"per_player"}, \code{"universal_constants"}) are
  not yet implemented.
```

**`@section Warnings:`** (currently at lines 137–148):

Replace the entire `@section Warnings:` block with:

```
@section Warnings:

\code{sgp()} may emit one or both of the following warnings via
\code{\link[cli]{cli_warn}}:

\subsection{rotostats_warning_missing_category_column}{
  A scored category column is entirely absent from \code{projections} —
  the affected \code{sgp_<CAT>} column is filled with \code{NA} for all
  players. Emitted once per missing category (Step 8).
}

\subsection{rotostats_warning_zero_playing_time}{
  A player has 0 or \code{NA} projected IP (when ERA or WHIP is scored) or
  projected AB (when AVG is scored) — that player's rate-stat SGP is set to
  \code{NA}. Emitted at Steps 14a, 14c, and 14d respectively. The message
  names the affected player(s).
}

To suppress only one class, use
\code{withCallingHandlers(rotostats_warning_missing_category_column = ...)}
or \code{withCallingHandlers(rotostats_warning_zero_playing_time = ...)}
independently.
```

**`@details` Validation order block** (currently at lines 58–70):

Insert a new item before item 1, renumbering the existing items. Replace the current 5-item list with this 6-item list:

```
## Validation order

Checks are applied in this order so that the most specific error wins:
1. \code{pool_baseline} must be \code{"projection_pool"} (else
   \code{rotostats_error_invalid_pool_baseline}). This check fires before
   any other validation, regardless of \code{rate_conversion}.
2. \code{rate_conversion} must be one of the five recognized values (else
   \code{rotostats_error_invalid_rate_conversion}).
3. When \code{rate_conversion = "blended_pool"}, \code{attr(denominators,
   "rate_conversion")} must also be \code{"blended_pool"} — mismatched units
   abort with \code{rotostats_error_invalid_rate_conversion}.
4. \code{"per_player"}, \code{"universal_constants"},
   \code{"team_ip_normalized"} abort with
   \code{rotostats_error_not_implemented}.
5. \code{"fixed_baseline"} delegates to \code{convert_rate_stats()} (stub;
   also aborts with \code{rotostats_error_not_implemented}).
6. Required inputs (\code{league_history}, \code{league_config}) are
   validated only after the method is confirmed.
```

---

## 3. Ordered edits to plans/error-messages.md

All three edits are surgical. Do not remove or reorder any existing rows.

### Edit 1 — Errors table: add new row

Append after the `rotostats_error_not_implemented` row:

```
| `rotostats_error_invalid_pool_baseline` | `sgp()` | `pool_baseline` is not one of the implemented values; currently only `"projection_pool"` is supported | Pass `pool_baseline = "projection_pool"` (the default) |
```

### Edit 2 — Warnings table: narrow existing `rotostats_warning_missing_category_column` row

Find the row:

```
| `rotostats_warning_missing_category_column` | `sgp()` | A scored category in `names(denominators)` is absent from `projections`, OR a player has 0 or NA projected IP/AB for a scored rate stat; the affected `sgp_<cat>` values are set to `NA`. Both the "column entirely absent" and the "zero playing time" cases use this class — distinguish them by reading the message text | Add the missing column to `projections`, or remove the category from the scored list; for zero playing time, the player legitimately has no rate-stat contribution |
```

Replace with:

```
| `rotostats_warning_missing_category_column` | `sgp()` | A scored category in `names(denominators)` is entirely absent from `projections`; the affected `sgp_<cat>` column is filled with `NA` for all players | Add the missing column to `projections`, or remove the category from the scored list |
```

### Edit 3 — Warnings table: add new `rotostats_warning_zero_playing_time` row

Insert immediately after the narrowed `rotostats_warning_missing_category_column` row:

```
| `rotostats_warning_zero_playing_time` | `sgp()` | A player has 0 or `NA` projected IP (when ERA or WHIP is scored) or 0 or `NA` projected AB (when AVG is scored); that player's rate-stat SGP is set to `NA` | This is expected for pure hitters (zero IP) or pitchers with no AB; suppress with `withCallingHandlers(rotostats_warning_zero_playing_time = ...)` if intentional |
```

---

## 4. Ordered edits to ARCHITECTURE.md

### Edit 1 — Diagram label at line ~127

Find:

```
    F --> F1["cli_warn\nrotostats_warning_missing_category_column"]
```

Replace with:

```
    F --> F1["cli_warn\nrotostats_warning_missing_category_column"]
    J --> J5a["cli_warn\nrotostats_warning_zero_playing_time"]
```

Note: `J` is the "Steps 13–14: compute SGP columns" node in the existing diagram. Line 127's `F1` is correct for Step 8 (missing column, node F = "Step 8: warn missing columns"). The zero-playing-time warn fires inside Steps 14a/14c/14d, which are inside node J. The existing `J --> J5["cli_warn zero IP/AB"]` node at line 143 should be updated to read:

```
    J --> J5["cli_warn\nrotostats_warning_zero_playing_time\nzero IP/AB"]
```

### Edit 2 — KDD item 3 at line ~448

Find the prose block:

```
3. **`rotostats_warning_missing_category_column` dual use**: The same class covers "column absent from projections" (Step 8) and "player has 0 or NA playing time" (Steps 14a, 14d). Messages are distinguishable by content; one class makes it easy to suppress both with a single `withCallingHandlers` call.
```

Replace with:

```
3. **Warning class split — `rotostats_warning_missing_category_column` vs `rotostats_warning_zero_playing_time`**: Two distinct warning classes replace the former dual-use design. `rotostats_warning_missing_category_column` fires at Step 8 when a scored category column is entirely absent from `projections`. `rotostats_warning_zero_playing_time` fires at Steps 14a, 14c, and 14d when a player has 0 or NA projected IP/AB for a scored rate stat. Callers may suppress either class independently via `withCallingHandlers`.
```

---

## 5. NEWS.md entry

Add the following bullets under the existing `## Implementation notes for maintainers` section, or as a new `## Breaking changes` section if that is clearer. Match the existing style (asterisk bullets, backtick code, plain prose).

**Preferred placement:** Insert before the `## Implementation notes for maintainers` section, as a new `## Breaking changes` section header:

```markdown
## Breaking changes

* `sgp()` now validates `pool_baseline` at the top of the function.
  Passing `pool_baseline = "per_player"` or
  `pool_baseline = "universal_constants"` now aborts immediately with
  `rotostats_error_invalid_pool_baseline` rather than propagating to a
  different downstream error. Only `pool_baseline = "projection_pool"`
  (the default) is accepted.

* `sgp()` now emits `rotostats_warning_zero_playing_time` (instead of
  `rotostats_warning_missing_category_column`) when a player has 0 or `NA`
  projected IP or AB for a scored rate stat. Callers using
  `withCallingHandlers(rotostats_warning_missing_category_column = ...)` to
  intercept zero-playing-time rows must update to
  `withCallingHandlers(rotostats_warning_zero_playing_time = ...)`.
  The condition triggering `rotostats_warning_missing_category_column` is
  unchanged: it fires only when a scored category column is entirely absent
  from `projections`.
```

---

## 6. Input validation contract

The `sgp()` argument list does NOT change. Only the function body and documentation change.

**New validation step summary:**

| Step | Check | Class on failure |
|------|-------|-----------------|
| 1b (new) | `pool_baseline %in% "projection_pool"` | `rotostats_error_invalid_pool_baseline` |

**Call-site class summary (warnings only, changes highlighted):**

| Line | Step | Condition | Class |
|------|------|-----------|-------|
| 342 | 8 | Scored category column absent | `rotostats_warning_missing_category_column` (unchanged) |
| 539 | 14a | ERA: player has 0 or NA IP | **`rotostats_warning_zero_playing_time`** (changed) |
| 572 | 14c | WHIP: player has 0 or NA IP | **`rotostats_warning_zero_playing_time`** (changed) |
| 603 | 14d | AVG: player has 0 or NA AB | **`rotostats_warning_zero_playing_time`** (changed) |

---

## 7. Explicit non-goals

The following MUST NOT change:

- The argument list for `sgp()` — no new parameters.
- The SGP math (blended-pool formulas for ERA, WHIP, AVG; counting-stat division) — unchanged.
- The message text of any `cli::cli_warn()` call — only the `class =` argument changes at lines 539, 572, 603.
- The `cli::cli_warn()` at line 342 — message AND class unchanged.
- `rate_conversion` validation (Steps 2, 3, 4, 5) — unchanged.
- Step 6 (`league_history` / `league_config` validation) — unchanged.
- Steps 7–15 logic, except for the three `class =` argument flips — unchanged.
- `sgp_denominators()`, `league_history()`, `league_config()`, `replacement_level()` — not touched.
- Any test file — builder does not touch test files (that is tester's write surface).
- `ARCHITECTURE.md` diagram nodes other than the ones identified in §4 — unchanged.
- `plans/error-messages.md` rows other than the three identified in §3 — unchanged.

---

## 8. Implementation notes (r-package profile)

- Use `cli::cli_abort()` for errors; `cli::cli_warn()` for warnings. Never `stop()` or `warning()`.
- Run `devtools::document()` after roxygen edits to regenerate `man/sgp.Rd`. Scriber owns this step.
- The `class =` argument to `cli::cli_abort()` and `cli::cli_warn()` must be a character scalar matching the class names in `plans/error-messages.md` exactly.
- No row-wise loops — keep all existing vectorized operations.
