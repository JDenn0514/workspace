<!-- filename: 2026-04-17-replacement-name-match-audit.md -->
# 2026-04-17 — Wire rotostats_warning_name_match_failure at Two Emit Sites

> Run: `replacement-name-match-audit-2026-04-17` | Profile: r-package | Verdict: PASS

## What Changed

The warning class `rotostats_warning_name_match_failure` was registered in `plans/error-messages.md` (line 96) and documented in roxygen comments at `R/replacement.R:960-961,973`, but grep of `R/` returned zero hits — no `cli_warn()` call with this class existed anywhere in the package. The helper `normalize_player_name()` (defined in `R/replacement_internal.R:746`) that was intended to power the matching also had zero callers.

This run wires two `cli_warn()` emission sites into `R/replacement.R`: one inside `.validate_league_history_inputs()` (Site 1, cross-reference check) and one inside `replacement_from_prices()` (Site 2, self-deduplication collision check). Both are gated by `verbose = TRUE` and are purely diagnostic — no data is filtered or modified by either warning. Four tests (plus one calibration invariance test) are added or replaced to assert the registered class fires correctly and does not fire spuriously. This run closes item R1 from `plans/replacement-cleanup.md`.

## Files Changed

| File | Action | Description |
|---|---|---|
| `R/replacement.R` | Modified | Two `cli_warn()` emit blocks added: Site 1 at lines 1271–1290 (inside `.validate_league_history_inputs()`), Site 2 at lines 1045–1070 (inside `replacement_from_prices()`); two em-dash literals replaced with `\u2014` in builder respawn cycle 1 |
| `NEWS.md` | Modified | One user-visible entry added under `## Improvements` |
| `tests/testthat/test-replacement-from-prices.R` | Modified | TS-49 replaced with T-NMF-1 (real warning assertion); T-NMF-1b (calibration invariance), T-NMF-3 (gate test), T-NMF-4 (no spurious emission) added |
| `tests/testthat/test-replacement.R` | Modified | T-NMF-2 appended under new `# § Name Match Failure Warning Tests` section header |

## Process Record

### Proposal (from planner)

**Implementation spec summary (from spec.md):**

- Two `cli_warn()` call sites to be inserted into existing function bodies in `R/replacement.R`. No new functions, no signature changes, no NAMESPACE changes, no new package dependencies.
- **Site 1** — inside the existing `if (!is.null(league_history$prices))` block in `.validate_league_history_inputs()`, inserted after the required-columns abort check. Gate: `if (verbose)` outer, `if (length(unmatched_names) > 0L)` inner. Algorithm: `normalize_player_name()` on both `prices_df$PLAYER_NAME` and `projections$PLAYER_NAME`, then `setdiff()`. Sample up to 3 unmatched names in the message.
- **Site 2** — inside `replacement_from_prices()`, inserted after column upcasing (line 1042/1043) and before the `§8 Algorithm` comment. Gate: `if (verbose && !("PLAYER_ID" %in% names(prices)))`. Algorithm: `normalize_player_name()` on all `prices$PLAYER_NAME` rows, then `split()` + `vapply()` to find normalized keys mapping to more than one distinct raw spelling. Sample up to 3 collision keys in the message.
- Critical design choice: Site 2 collision check operates on full `prices` (post-upcasing, pre-`dollar_pool` filter at line 1050) for maximum diagnostic coverage.
- Message copy was fully specified in spec.md §5. Both messages include the diagnostic disclaimer: `"This is a diagnostic warning only \u2014 ..."`.
- One NEWS.md entry specified in spec.md §7.

**Test spec summary (from test-spec.md):**

- **T-NMF-1 (replaces TS-49)**: `replacement_from_prices()` with `verbose = TRUE`, `player_id` absent, prices containing `"Jose Ramirez"` and `"Jos\u00e9 Ram\u00edrez"` (deterministic collision) → `expect_warning(class = "rotostats_warning_name_match_failure")`.
- **T-NMF-1b**: Calibration invariance — `replacement_stats` output identical whether using collision fixture or clean fixture with identical stat values (`tolerance = 1e-9`).
- **T-NMF-2**: `replacement_level()` with `league_history$prices` containing `"Z\u00e9 Sil\u00e4o"` (not in projections) and `verbose = TRUE` → warning fires.
- **T-NMF-3**: Same collision fixture with `verbose = FALSE` → no warning of this class.
- **T-NMF-4**: Clean all-ASCII unique names with `verbose = TRUE` → no spurious warning.
- Static grep checks: (1) class string count >= 2 in `R/replacement.R`; (2) `normalize_player_name` call count >= 2; (3) no `stri_trans_nfd` or `\p{Mn}` in `R/replacement.R`.
- Full suite: `devtools::document()` clean; `devtools::test()` FAIL=0 no new SKIPs; `R CMD check` 0 ERRORs 0 WARNINGs NOTEs unchanged from develop baseline.

### Implementation Notes (from builder)

- Builder commit `80afe5c`: both emit sites implemented verbatim per spec.md §3 and §4. No deviations from spec.
- `devtools::document()` ran clean; no unexpected `man/` or NAMESPACE changes (roxygen at lines 960-961 and 973 already documented the intended behavior).
- Grep verification: `rotostats_warning_name_match_failure` hit count = 2 (lines 1067 and 1287); `normalize_player_name` hit count = 3 (lines 1047, 1272, 1273). No forbidden normalization patterns.
- **Builder respawn cycle 1 (commit `272cb97`)**: Triggered by tester BLOCK (see Problems Encountered). Fix: replaced two raw em-dash bytes (U+2014, `—`) in `cli_warn()` string literals with the R escape sequence `\u2014`. Line 1065 (Site 2) and line 1285 (Site 1). Runtime message is byte-identical; source file is now ASCII-clean.
- Worktree note: the worktree branch was fast-forwarded from develop tip (`240f683`) before implementation — it was missing `R/replacement.R`. Feature work sits cleanly on top of develop.
- No new imports: `split()`, `vapply()`, `unique()`, `setdiff()`, `head()` are base R; `normalize_player_name()` uses `stringi` which is already in `Imports`.
- Tests written by tester from `test-spec.md` in tester commit `d9b9fe8`. Tester did not need to re-commit in cycle 1 — all test files were complete from cycle 0.

### Validation Results (from tester)

**Per-Test Result Table (from audit.md):**

| Test ID | File | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|---|---|---|---|---|---|---|---|
| T-NMF-1 (TS-49) | test-replacement-from-prices.R | Warning class emitted on collision fixture | `rotostats_warning_name_match_failure` | Warning of class `rotostats_warning_name_match_failure` fired | exact class match | — | PASS |
| T-NMF-1b | test-replacement-from-prices.R | `replacement_stats$HR` identical (collision vs clean) | same value | same value | atol=1e-9 | 0% | PASS |
| T-NMF-1b | test-replacement-from-prices.R | `replacement_stats$R` identical | same value | same value | atol=1e-9 | 0% | PASS |
| T-NMF-1b | test-replacement-from-prices.R | `replacement_stats$RBI` identical | same value | same value | atol=1e-9 | 0% | PASS |
| T-NMF-1b | test-replacement-from-prices.R | `replacement_stats$SB` identical | same value | same value | atol=1e-9 | 0% | PASS |
| T-NMF-1b | test-replacement-from-prices.R | `replacement_stats$position` identical | same value | same value | exact | — | PASS |
| T-NMF-2 | test-replacement.R | Warning class emitted on unmatched prices name | `rotostats_warning_name_match_failure` | Warning of class `rotostats_warning_name_match_failure` fired | exact class match | — | PASS |
| T-NMF-3 | test-replacement-from-prices.R | No warning class when verbose=FALSE | no `rotostats_warning_name_match_failure` | no warning of that class | exact absence | — | PASS |
| T-NMF-4 | test-replacement-from-prices.R | No warning class when names all distinct | no `rotostats_warning_name_match_failure` | no warning of that class | exact absence | — | PASS |

**Summary:** 9 test-metric rows executed across 5 test scenarios. All passed. FAIL=0 in full suite (`[ FAIL 0 | WARN 126 | SKIP 1 | PASS 479 ]`). SKIP 1 is pre-existing (TS-27, dead-code guard). WARN 126 are side-effect warnings emitted by functions under test — expected, count unchanged from baseline.

**Before/After Comparison Table (from audit.md):**

| Metric | Before (develop) | After (feature) | Change | Interpretation |
|---|---|---|---|---|
| `replacement_from_prices()` emits `rotostats_warning_name_match_failure` on collision | Never (class not wired) | Always when `verbose=TRUE` and `player_id` absent | New behavior | Expected improvement — warning now fires as documented |
| `replacement_level()` emits `rotostats_warning_name_match_failure` on unmatched prices | Never (class not wired) | Always when `verbose=TRUE` and unmatched names exist | New behavior | Expected improvement |
| `replacement_stats` output (stat values) | Unchanged | Unchanged | 0 | Diagnostic-only — confirmed by T-NMF-1b at atol=1e-9 |
| R CMD check WARNINGs | 0 | 0 | 0 | Resolved — was 1 in cycle 0 (non-ASCII); fixed in `272cb97` |
| R CMD check NOTEs | 4 | 4 | 0 | Unchanged — same 4 pre-existing NOTEs as develop baseline |
| `checking code files for non-ASCII characters` | OK | OK | None | Builder ASCII fix (`272cb97`) resolved the cycle-0 WARNING |

**Validation commands:**
- `devtools::document()`: clean output, no unexpected man/ or NAMESPACE changes.
- `devtools::test()`: `[ FAIL 0 | WARN 126 | SKIP 1 | PASS 479 ]` — all five T-NMF scenarios pass.
- `devtools::check(args = "--no-manual", error_on = "warning")`: `0 errors | 0 warnings | 4 notes`. All 4 NOTEs pre-existing (`.playwright-mcp` hidden dir, non-standard top-level files, NEWS.md format, Rd escaped LaTeX specials).
- Static greps: class string count = 2; `normalize_player_name` count = 3; no `stri_trans_nfd` or `\p{Mn}` in `R/replacement.R`.
- `tools::showNonASCIIfile("R/replacement.R")`: all 32 flagged lines are comments (`#` or `#'`). Lines 1065 and 1285 (the previously offending `cli_warn()` literals) are not in the list.

### Problems Encountered and Resolutions

| # | Problem | Signal | Routed To | Resolution |
|---|---|---|---|---|
| 1 | Builder commit `80afe5c` introduced raw em-dash bytes (U+2014, `—`) as literal UTF-8 in two `cli_warn()` string literals in `R/replacement.R`. R CMD check raised `checking code files for non-ASCII characters ... WARNING`, causing the R CMD check acceptance criterion to fail. Note: roxygen comment em-dashes (already present in the file) are NOT flagged — R CMD check only checks non-comment code. | BLOCK | Builder | Builder respawn cycle 1 (commit `272cb97`): replaced `—` with `\u2014` escape at lines 1065 and 1285. Source file is now ASCII-clean; rendered warning message is byte-identical at runtime. Tester re-ran `devtools::check()` in cycle 1 and confirmed `0 errors | 0 warnings | 4 notes`. |

**Signal history from mailbox.md:**
- Planner handoff: builder and tester instructions routed via mailbox.md. No HOLD conditions. Comprehension verdict: FULLY UNDERSTOOD.
- Builder handoff (`80afe5c`): no blockers, no interface changes, static verification passed.
- Tester BLOCK: `R CMD check WARNING` — non-ASCII characters in R code string literals. Routed to builder.
- Builder respawn cycle 1 (`272cb97`): ASCII fix applied. No further signals.
- Tester cycle 1 audit: PASS. All acceptance criteria met.

### Review Summary

Pending — reviewer review follows scriber.

- **Pipeline isolation**: pending
- **Convergence**: pending
- **Tolerance integrity**: pending
- **Verdict**: pending

## Design Decisions

1. **`verbose = TRUE` gate for both emit sites**: The warning class was registered in `plans/error-messages.md` line 96 with the note "only fires when `verbose = TRUE`". This gate is the registered behavior — it ensures that diagnostic noise does not appear in default (silent) usage. The gate is consistent with all other `verbose`-gated warnings in the replacement module (e.g., `rotostats_warning_team_total_divergence` at line 95 of the same registry).

2. **`\u2014` escape over plain ASCII hyphen**: The spec called for an em-dash in the diagnostic disclaimer string `"This is a diagnostic warning only — ..."`. The initial implementation used a raw UTF-8 em-dash, which is valid in R source encoding (UTF-8) but triggers an R CMD check WARNING for non-ASCII in R code. The fix uses `\u2014`, which R evaluates to the same Unicode character at runtime. A plain ASCII hyphen (`-`) would have changed the rendered output and was rejected. The `\u2014` escape preserves the exact rendered message while keeping the source file ASCII-clean — the correct approach for any em-dash in R string literals.

3. **Site 2 collision check on full `prices` (pre-`dollar_pool` filter)**: The collision detection in `replacement_from_prices()` runs on all rows of `prices` after column upcasing, not on the `dollar_pool` subset (price <= 1). This gives maximum diagnostic coverage — name collisions among $2 or higher price tiers are surfaced even though those rows do not participate in calibration. The diagnostic intent of `verbose = TRUE` is to surface data-quality issues; filtering to `dollar_pool` before the check would hide real collisions from the user.

4. **`normalize_player_name()` as the single normalization authority**: The spec and hard constraints explicitly prohibit introducing a duplicate normalization scheme (no new `stri_trans_nfd`, `tolower`, or `gsub("[^a-z...]")` calls). Both emit sites delegate entirely to `normalize_player_name()` from `R/replacement_internal.R`. This centralizes the canonical definition of "same player" and makes it easy to update normalization behavior in one place if needed.

5. **`split()` + `vapply()` for collision detection (no row-wise loops)**: The spec required that both emit sites be vectorized, with no row-wise loops. Site 2 uses `split(raw_names, norm_names)` to group raw spellings by normalized key, then `vapply()` to count distinct raw spellings per group. This is fully vectorized and efficient even for large prices data frames.

## Handoff Notes

- This run closes item R1 from `plans/replacement-cleanup.md`. No follow-up items from this run.
- The `rotostats_warning_name_match_failure` class is now live at both documented sites. If the `normalize_player_name()` helper is ever updated (e.g., to handle different Unicode normalization forms), both emit sites will automatically pick up the change — no modification to `replacement.R` is needed.
- The NEWS.md `## Improvements` section added in commit `80afe5c` does not resolve the pre-existing `checking package subdirectories ... NOTE: Problems with news in 'NEWS.md': No news entries found` R CMD check NOTE. That NOTE is a pre-existing format issue unrelated to this change.
- The `warning_class` registry in `plans/error-messages.md` now reflects the actual behavior: `rotostats_warning_name_match_failure` is wired and fires. The "Thrown by" column at line 96 reads `replacement_level() (league history validation, name normalization)`. Consider updating it to also list `replacement_from_prices()` as a throwing function in a follow-up docs pass, since Site 2 is now also live.
- No changes to `NAMESPACE`, `man/`, `DESCRIPTION`, or any function signature. The PR requires no breaking-change annotations.
