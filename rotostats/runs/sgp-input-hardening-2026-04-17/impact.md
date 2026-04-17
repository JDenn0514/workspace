# Impact — sgp-input-hardening-2026-04-17

## Profile

`r-package` (see `profiles/r-package.md`). Target repo: `/Users/jacobdennen/rotostats`.

## Affected files (target repo)

### Source

- `R/sgp.R`
  - Function body: add `pool_baseline` top-of-function validation (a new step between existing Step 1 and Step 2).
  - Call site at `R/sgp.R:342` — Step 8 (**column absent** condition). Keep class `rotostats_warning_missing_category_column`; keep message intact. **No change to class.**
  - Call site at `R/sgp.R:539` — Step 14a ERA zero-IP. **Change class** to `rotostats_warning_zero_playing_time`.
  - Call site at `R/sgp.R:572` — Step 14c WHIP zero-IP. **Change class** to `rotostats_warning_zero_playing_time`.
  - Call site at `R/sgp.R:603` — Step 14d AVG zero-AB. **Change class** to `rotostats_warning_zero_playing_time`.
  - Roxygen `@section Warnings:` block at `R/sgp.R:137–148` — split into the two classes.
  - Roxygen `@param pool_baseline` at `R/sgp.R:107–109` — document that any value other than `"projection_pool"` aborts.
  - `@details` Validation order block (`R/sgp.R:58–70`) — add a new item noting `pool_baseline` check placement.

### Generated docs (rebuilt via `devtools::document()`)

- `man/sgp.Rd` — regenerates from the updated roxygen. Scriber owns this regeneration.

### Package metadata

- `NEWS.md` — add a dev version entry noting the minor breaking change to the
  warning class split (callers using `withCallingHandlers()` on
  `rotostats_warning_missing_category_column` for zero-playing-time rows now
  receive `rotostats_warning_zero_playing_time`).

### Project-level docs / plans

- `plans/error-messages.md`
  - Errors table: add `rotostats_error_invalid_pool_baseline` row (thrown by `sgp()`).
  - Warnings table: narrow description of `rotostats_warning_missing_category_column` to the column-absent case only. Add new row for `rotostats_warning_zero_playing_time`.

- `ARCHITECTURE.md`
  - Diagram label at line ~127: `F --> F1["cli_warn\nrotostats_warning_missing_category_column"]` — add a second branch / second node for the zero-playing-time class.
  - Prose block at line ~448 (item 3 under "Warning class philosophy" or similar) — replace the "dual use" explanation with a sentence describing the split.

### Tests (tester write surface — reached via test-spec.md)

- `tests/testthat/test-sgp.R`
  - `R/.../test-sgp.R:474–486` ("missing scored category column" test) — still expects `rotostats_warning_missing_category_column`. No change.
  - `R/.../test-sgp.R:488–500` ("zero IP player" test for ERA) — now expects `rotostats_warning_zero_playing_time`. Update.
  - `R/.../test-sgp.R:520–528` (third expectation) — check whether it's a zero-playing-time case; if so, update class. Tester inspects.
  - Add three new tests: (a) `pool_baseline = "foo"` aborts with `rotostats_error_invalid_pool_baseline`; (b) missing category fires missing-column class and NOT zero-playing-time; (c) zero IP fires zero-playing-time class and NOT missing-column.

- `tests/testthat/test-sgp-integration.R`
  - Three call sites at lines 520, 572, 625 assert the old class. Tester reviews each and reassigns to whichever class now matches the condition under test.

## Risk areas

- **Back-compat for warning handlers.** Users calling `withCallingHandlers(rotostats_warning_missing_category_column = ...)` on zero-playing-time rows will no longer see those warnings under the old class. Mitigated by `NEWS.md` entry; called out as a minor breaking change.
- **Doc regeneration.** Scriber must run `devtools::document()` so `man/sgp.Rd` stays consistent with roxygen. Otherwise R CMD check raises a NOTE.
- **`plans/error-messages.md` dual edit.** Planner MUST NOT silently drop any of the many existing entries when editing the warnings table — only surgical narrowing of one row plus one new row.

## Risk non-areas (explicitly out of scope)

- **No algorithmic change.** SGP math is unchanged.
- **No behavioral change for `pool_baseline = "projection_pool"`.** The default path is identical.
- **No change to `rate_conversion` validation**, `league_config` validation, or any other existing check.
- **No change to other warning/error classes** beyond the three listed.
- **No change to `sgp_denominators()`, `league_history()`, `league_config()`, or `replacement_level()`.**

## Required teammates

- Planner (MANDATORY).
- Builder (worktree) — edits `R/sgp.R` only.
- Tester — runs `devtools::test()`, `devtools::check()`; validates the three new behaviors.
- Scriber (worktree) — regenerates `man/sgp.Rd` via `devtools::document()`; updates `NEWS.md`, `ARCHITECTURE.md`, `plans/error-messages.md`; writes `docs.md` and `log-entry.md`.
- Reviewer.
- Shipper (workflow 2 — user asked to ship and the plan targets a PR to `develop`).

No simulator (input-validation change, not statistical). No distiller (brain mode isolated).

## Workflow gate decisions

- **Simplified workflow?** No. Although the change is surgical (touches 3 files + 1 generated file + tests + docs), it adds a new error class and changes the public warning surface, which warrants planner's comprehension + spec contract. Use the full workflow.
- **Uploaded files?** None.
- **Brain dispatch?** No — isolated mode.

## State transition plan

```
NEW → CREDENTIALS_VERIFIED (done) → PLANNED → SPEC_READY → PIPELINES_COMPLETE → DOCUMENTED → REVIEW_PASSED → READY_TO_SHIP → DONE
```
