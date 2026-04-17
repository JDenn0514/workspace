# Mailbox — sgp-input-hardening-2026-04-17

Teammates append messages here to raise blockers, interface changes, or cross-cutting concerns that leader needs to see.

---

## Planner handoff — 2026-04-17

**Status:** SPEC_READY. All three artifacts written. Comprehension verdict: FULLY UNDERSTOOD. Zero HOLD rounds.

### For builder

Read `spec.md` in full. The two changes are surgical:

1. Insert a new Step 1b validation block (6 lines of R) between lines 219 and 222 of `R/sgp.R`. The block checks `pool_baseline %in% "projection_pool"` and aborts with `rotostats_error_invalid_pool_baseline`. See spec.md §2a for the exact code expression.

2. Change the `class =` argument (and only that) at three existing `cli::cli_warn()` call sites — lines 539, 572, 603 — from `"rotostats_warning_missing_category_column"` to `"rotostats_warning_zero_playing_time"`. Line 342 is unchanged.

3. Update roxygen: `@param pool_baseline`, `@section Warnings:`, and the `@details Validation order` block. See spec.md §2c for the full replacement text of each block.

4. Make three surgical edits to `plans/error-messages.md`: one new error row, one narrowed warning row, one new warning row. See spec.md §3 for exact row text.

5. Make two surgical edits to `ARCHITECTURE.md`: update the diagram node for J5, and replace KDD item 3 prose. See spec.md §4.

6. Add `## Breaking changes` section to `NEWS.md` with two bullets. See spec.md §5 for exact text.

Do NOT touch test files, do NOT change the argument list of `sgp()`, do NOT alter any SGP math.

### For tester

Read `test-spec.md` in full. Key actions:

1. Update `test-sgp.R:488` — change expected class from `rotostats_warning_missing_category_column` to `rotostats_warning_zero_playing_time`. See test-spec.md §4a.

2. Update `test-sgp.R:517–527` — same class change for the zero-AB AVG test. See test-spec.md §4b.

3. Integration tests: line 520 (TS-10) is condition (a) — leave class unchanged. Lines 572 (TS-11) and 625 (TS-12) are condition (b) — change to `rotostats_warning_zero_playing_time`. See test-spec.md §5.

4. Add three new tests to `test-sgp.R` Section 6 (or new Section 8): (a) invalid `pool_baseline` aborts, (b) missing column fires only missing-column class, (c) zero IP fires only zero-playing-time class. Also add two supplementary tests for `"per_player"` and `"universal_constants"`. See test-spec.md §3.

5. Run `devtools::test(filter = "sgp")` and `devtools::check()`. Compare NOTE count on `develop` HEAD vs feature branch.

### For scriber

After builder's commit: run `devtools::document()` to regenerate `man/sgp.Rd`. Verify the roxygen changes in spec.md §2c are reflected in the generated Rd. The `NEWS.md` and `ARCHITECTURE.md` edits may be made in the same pass — they are specified in spec.md §4 and §5.

---

## Builder handoff — 2026-04-17

**Status:** IMPLEMENTATION_COMPLETE. Commit `a1a56e1` on `feature/sgp-input-hardening`.

### Interface changes for teammates

- **New error class** `rotostats_error_invalid_pool_baseline` emitted by `sgp()` when `pool_baseline` is not `"projection_pool"`. Registered in `plans/error-messages.md`.
- **Warning class split**: `rotostats_warning_zero_playing_time` is now a distinct class (previously subsumed under `rotostats_warning_missing_category_column`). Call sites at old lines 539/572/603 now emit the new class. Line 342 (column-absent case, Step 8) is unchanged.
- **`man/sgp.Rd` not regenerated**: Scriber must run `devtools::document()`. The roxygen source in `R/sgp.R` is complete.

### No blockers.

All four files within the write surface were successfully modified and committed. No ambiguities encountered; spec.md was fully sufficient.

---

## Scriber handoff — 2026-04-17

**Status:** DOCUMENTATION_COMPLETE. Commit `240f683` on `develop` (target checkout was on develop; feature/sgp-input-hardening-docs had already merged via PR #5).

### What scriber did

- Ran `devtools::document()` in `/Users/jacobdennen/rotostats`: only `man/sgp.Rd` was regenerated. NAMESPACE unchanged.
- Ran `devtools::check(document = FALSE, args = c('--no-manual'))`: `0 errors | 0 warnings | 5 notes` — identical to tester baseline.
- Updated `ARCHITECTURE.md` header and diagram highlights for this run.
- Wrote `docs.md` and `log-entry.md` to run directory.
- Copied `ARCHITECTURE.md` to run directory.

### For reviewer

- Commit to review: `240f683` (on develop, 1 ahead of origin/develop).
- `man/sgp.Rd` now correctly documents `pool_baseline` abort behavior, 6-item validation order, and two-class warning section.
- No other Rd files were touched by `devtools::document()`.
- `ARCHITECTURE.md` diagrams and KDD content were written by builder in `a1a56e1`; scriber updated only header metadata and highlight styles.

### No blockers.
