<!-- filename: 2026-04-21-inverse-categories-infrastructure.md -->

# 2026-04-21 — Inverse-Categories Infrastructure

> Run: `inverse-categories-2026-04-21` | Profile: r-package | Verdict: PASS

## What Changed

Added a three-layer inverse-category resolution infrastructure to rotostats. A new package-level lookup constant (`INVERSE_CATEGORIES`) and exported accessor (`inverse_categories()`) replaces the old hardcoded `c("ERA","WHIP")` default in `sgp_denominators()`. `league_config()` gains an optional `inverse_categories` field so directionality can be declared once at the league level. `sgp_denominators()` now resolves the effective inverse set in priority order: (1) explicit user arg (full replacement), (2) `config$inverse_categories`, (3) `intersect(scoring_categories, inverse_categories())` — preserving the ERA/WHIP legacy behavior unchanged when those categories are scored.

## Files Changed

| File | Action | Description |
| --- | --- | --- |
| `R/inverse-categories.R` | created | `INVERSE_CATEGORIES` internal constant (8-element) + `inverse_categories()` exported accessor |
| `R/league-config.R` | modified | New `inverse_categories = NULL` param, `validate_inverse_categories()` helper, `$inverse_categories` S3 slot, `Inverse:` print line |
| `R/sgp-denominators.R` | modified | Default changed from `c("ERA","WHIP")` to `NULL`; new `config = NULL` arg; three-layer resolution block replaces old ~55-line block; removed `missing()` flag |
| `specs/spec-sgp.md` | modified | `inverse_categories` param row updated (default + description); new `config` param row added in `sgp_denominators()` section only |
| `plans/error-messages.md` | modified | `rotostats_error_invalid_inverse_categories` "Thrown by" column updated to include `league_config()` |
| `NEWS.md` | modified | New bullet under Unreleased `## New arguments` section |
| `NAMESPACE` | auto-generated | `export(inverse_categories)` added |
| `man/inverse_categories.Rd` | auto-generated | New man page for `inverse_categories()` |
| `man/league_config.Rd` | auto-generated | Updated with `inverse_categories` param |
| `man/sgp_denominators.Rd` | auto-generated | Updated with revised `inverse_categories` param + new `config` param |
| `tests/testthat/test-inverse-categories.R` | created | 4 tests: TC-INV-1 through TC-INV-4 |
| `tests/testthat/test-league-config.R` | modified | 7 tests added: TC-LC-INV-1 through TC-LC-INV-7 |
| `tests/testthat/test-sgp-denominators.R` | modified | 9 tests added: TC-SGP-INV-1 through TC-SGP-INV-9 |
| `ARCHITECTURE.md` | modified | Module structure + call graph + data flow + reference table updated for this run |

## Process Record

This section captures the full workflow history: what was proposed, what was tested, what problems arose, and how they were resolved.

### Proposal (from planner)

**Implementation spec summary** (from `spec.md`):

- **Core idea**: The package's directionality knowledge (which categories are lower-is-better) was embedded in a single hardcoded default `c("ERA","WHIP")` on `sgp_denominators()`. This plan lifts it to a proper package-level lookup with three access tiers.
- **Layer-1 resolution**: When the user passes an explicit `inverse_categories` arg to `sgp_denominators()`, it wins unconditionally (full replacement — config is never consulted). Validated against the effective scored-category set.
- **Layer-2 resolution**: When `inverse_categories = NULL` (default) but `config$inverse_categories` is non-NULL, the config's list is used verbatim (already validated and uppercased at construction time — no re-validation).
- **Layer-3 resolution**: Fallback: `intersect(scoring_categories, inverse_categories())` — the package-level lookup intersected with what is actually scored. Preserves the old ERA/WHIP behavior when those categories are scored.
- **Q1 design decision (config inheritance)**: Option (a) chosen — new `config = NULL` arg on `sgp_denominators()`. Options (b) (attach config to `league_history`) and (c) (punt Layer-2) were rejected: (b) violates `league_history`'s own design contract; (c) violates plan acceptance criterion §5.
- **Q2 cli_inform wording**: Three parenthetical source labels committed: `"(user override)"`, `"(from config)"`, `"(package default)"`. Zero-element case appends the same label.
- **Q3 starter list confirmed**: `c("ERA","WHIP","FIP","xFIP","SIERA","xERA","BB/9","HR/9")` — 8 elements. Mixed case `xFIP`/`xERA` is conventional FanGraphs notation.
- **Critical design rule**: `config` arg position in `sgp_denominators()` is after `inverse_categories`, before `rate_conversion`. Only `config$inverse_categories` is consulted — no other config fields.

**Test spec summary** (from `test-spec.md`):

- **20 new tests** across 3 test files: 4 in `test-inverse-categories.R`, 7 in `test-league-config.R` (additions), 9 in `test-sgp-denominators.R` (additions).
- **TC-INV-1 through TC-INV-4**: Verify the accessor returns a character vector, contains exactly the 8 starter elements (setequal + length + no-duplicates), is callable without error/warning, and does NOT contain `"K/9"`.
- **TC-LC-INV-1 through TC-LC-INV-7**: Verify NULL default storage, valid vector uppercasing + storage, abort on element not in categories, abort on non-character, print output for NULL and non-NULL cases, and S3 field accessibility.
- **TC-SGP-INV-1 through TC-SGP-INV-9**: Verify Layer-3 preserves ERA/WHIP behavior, Layer-3 correctly includes/excludes FIP based on scored categories, Layer-2 inherits from config, Layer-2 falls to Layer-3 when config$inverse_categories is NULL, Layer-1 provides full replacement (config ignored), Layer-1 aborts on unknown element, invalid config arg aborts, legacy back-compat.
- **Key test design constraint**: `cli_inform(.frequency = "once", .frequency_id = ...)` registers the frequency ID session-wide; tests using `withCallingHandlers` for message capture must use unique effective inverse sets so each captures a novel `.frequency_id`.
- **Property invariants**: P1 idempotency of uppercase, P2 full-replacement semantics, P3 monotone fallback, P4 Layer-3 subset of scoring_categories, P5 backwards compatibility.

### Implementation Notes (from builder)

- **Builder first pass** (commit `ef048ba`): Implemented all 6 surfaces from spec.md §11 in order. Zero deviations from spec except one: the spec draft `"element{?s}"` error message pattern triggered a cli runtime error ("Cannot pluralize without a quantity"). Fixed to `"{n_bad} element{?s} of {.arg inverse_categories}"` throughout — same as the existing pattern used elsewhere in the package. This fix resolved two pre-existing test failures (T-61 and T-70) that were already broken by the cli bug.
- **Builder Respawn-1 fix** (commit `00b68c8`): Applied the same `n_bad <- length(bad)` + `"{n_bad} element{?s} of {.arg inverse_categories} not in {.arg categories}."` pattern to `validate_inverse_categories()` in `R/league-config.R` (the bug tester F2 identified). This was the only builder-owned change in Respawn 1.
- **`missing()` flag removed**: The old `inverse_categories_is_default <- missing(inverse_categories)` at line 308 was eliminated; with `NULL` default, `!is.null(inverse_categories)` is equivalent and simpler.
- **Validator placement**: `validate_inverse_categories()` inserted immediately after `validate_categories()` in `R/league-config.R`, consistent with the file's organize-validators-by-constructor-order convention.
- **`inv_source` variable**: Holds the source label string assigned in each resolution branch; used in a single `cli_inform()` call block below the resolution logic.
- No test files written by builder (tests/testthat/ is tester's surface).
- `devtools::document()` run twice (second was clean with no cross-link warnings). `devtools::check()` from builder first pass: 0 errors, 0 warnings, 6 notes (all pre-existing).

### Validation Results (from tester)

**Initial tester run** (commit `232a82a`): BLOCK — 5 failures.

**Respawn-1 tester run** (commit `8efbc2c`): PASS — all 20 new tests pass.

**Per-Test Result Table** (Respawn-1 PASS run):

| Test ID | Metric / Assertion | Expected | Actual | Tolerance | Rel. Error | Verdict |
|---------|-------------------|----------|--------|-----------|------------|---------|
| TC-INV-1 | `is.character(result)` | TRUE | TRUE | exact | — | PASS |
| TC-INV-2 | `setequal(result, starter)` | TRUE | TRUE | exact | — | PASS |
| TC-INV-2 | `length(result)` | 8L | 8L | exact | — | PASS |
| TC-INV-2 | `length(result) == length(unique(result))` | TRUE | TRUE | exact | — | PASS |
| TC-INV-3 | `expect_no_error(inverse_categories())` | no error | no error | exact | — | PASS |
| TC-INV-3 | `expect_no_warning(inverse_categories())` | no warning | no warning | exact | — | PASS |
| TC-INV-4 | `"K/9" %in% inverse_categories()` | FALSE | FALSE | exact | — | PASS |
| TC-LC-INV-1 | `lg$inverse_categories` is NULL | NULL | NULL | exact | — | PASS |
| TC-LC-INV-2 | `lg$inverse_categories` | `c("ERA","WHIP","FIP")` | `c("ERA","WHIP","FIP")` | exact | — | PASS |
| TC-LC-INV-3 | error class | `rotostats_error_invalid_inverse_categories` | `rotostats_error_invalid_inverse_categories` | exact | — | PASS |
| TC-LC-INV-4 | error class | `rotostats_error_invalid_inverse_categories` | `rotostats_error_invalid_inverse_categories` | exact | — | PASS |
| TC-LC-INV-5 | output contains `"(none declared)"` | TRUE | TRUE | exact | — | PASS |
| TC-LC-INV-6 | output contains `"ERA"` | TRUE | TRUE | exact | — | PASS |
| TC-LC-INV-6 | output contains `"WHIP"` | TRUE | TRUE | exact | — | PASS |
| TC-LC-INV-6 | output does NOT contain `"(none declared)"` | FALSE | FALSE | exact | — | PASS |
| TC-LC-INV-7 | `is.character(lg$inverse_categories)` | TRUE | TRUE | exact | — | PASS |
| TC-LC-INV-7 | `setequal(lg$inverse_categories, c("FIP","ERA"))` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-1 | `is.numeric(d$denominators)` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-1 | `names(d$denominators)` | `{ERA, WHIP}` | `{ERA, WHIP}` | ignore.order | — | PASS |
| TC-SGP-INV-2 | any message contains `"package default"` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-2 | FIP absent from Effective inverse messages | FALSE | FALSE | exact | — | PASS |
| TC-SGP-INV-3 | any message contains `"package default"` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-3 | FIP present in Effective inverse message | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-4 | any message contains `"from config"` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-5 | any message contains `"package default"` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-6 | any message contains `"user override"` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-6 | FIP present in Effective inverse message | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-6 | ERA absent from Effective inverse message | TRUE (absent) | TRUE (absent) | exact | — | PASS |
| TC-SGP-INV-6 | WHIP absent from Effective inverse message | TRUE (absent) | TRUE (absent) | exact | — | PASS |
| TC-SGP-INV-7 | error class | `rotostats_error_invalid_inverse_categories` | `rotostats_error_invalid_inverse_categories` | exact | — | PASS |
| TC-SGP-INV-8 | error class | `rotostats_error_invalid_parameter` | `rotostats_error_invalid_parameter` | exact | — | PASS |
| TC-SGP-INV-9 | `is.numeric(d$denominators)` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-9 | `names(d$denominators)` | `{ERA, WHIP}` | `{ERA, WHIP}` | ignore.order | — | PASS |
| TC-SGP-INV-9 | `all(is.finite(d$denominators))` | TRUE | TRUE | exact | — | PASS |
| TC-SGP-INV-9 | `all(d$denominators > 0)` | TRUE | TRUE | exact | — | PASS |

**Summary: 0 FAIL, 20 new test scenarios PASS (35 individual assertions)**

**Before/After Comparison Table:**

| Metric | Before (develop) | After (feature) | Change | Interpretation |
| --- | --- | --- | --- | --- |
| `inverse_categories()` exported | does not exist | exists; returns 8-element mixed-case vector | new function | New infrastructure — consumers can query the package default |
| `league_config()` `$inverse_categories` slot | does not exist | NULL by default; validated, uppercased, stored | new field | New infrastructure — no breaking change |
| `sgp_denominators()` ERA/WHIP scoring, no arg | effective `{ERA,WHIP}` hardcoded | effective `{ERA,WHIP}` via Layer-3 intersect | identical output | Back-compat preserved |
| `sgp_denominators()` FIP scoring, no arg | FIP not direction-flipped (not in old hardcoded list) | FIP direction-flipped via Layer-3 | improved behavior | Correct for FIP leagues without user action |
| `sgp_denominators()` `config = NULL` arg | does not exist | accepted | new arg | No breaking change to existing callers |
| `print.league_config()` output | no Inverse line | shows `"Inverse: (none declared)"` or effective list | new line | Informational; no breaking change |

**Validation commands run (exact):**

```
devtools::test(filter = "inverse-categories")   # FAIL 0 | WARN 0 | SKIP 0 | PASS 7
devtools::test(filter = "league-config")         # FAIL 0 | WARN 7 | SKIP 0 | PASS 80
devtools::test(filter = "sgp-denominators")      # FAIL 0 | WARN 160 | SKIP 0 | PASS 179
devtools::check(error_on = "warning")            # 0 errors | 0 warnings | 7 notes
```

The 7 warnings in league-config are pre-existing "Unrecognized scoring category: FIP" from the `.minimal_config_args()` fixture. The 160 warnings in sgp-denominators are pre-existing calibration and data-quality warnings (thin windows, low R², unrecognized columns) — 3 more than the initial tester run's 157 due to the `.lh_era_whip_fip()` fixture used in the updated TC-SGP-INV-4 and TC-SGP-INV-5 generating expected FIP-related warnings.

**devtools::check() comparison:**

| Category | Baseline (develop) | Respawn-1 feature branch | Delta |
|----------|--------------------|--------------------------|-------|
| Errors | 0 | 0 | 0 |
| Warnings | 0 | 0 | 0 |
| Notes | 6 | 7 | +1 environmental ("unable to verify current time") |

The +1 note is environmental (clock verification), not caused by any code change.

**Frequency ID Uniqueness — all 6 message-capture tests confirmed novel at execution:**

| Test | Effective Set | `.frequency_id` | First registration? |
|------|---------------|---------------|---------------------|
| TC-SGP-INV-1 | {ERA, WHIP} | `rotostats_sgp_denom_inverse_ERA,WHIP` | YES |
| TC-SGP-INV-2 | {WHIP} | `rotostats_sgp_denom_inverse_WHIP` | YES |
| TC-SGP-INV-3 | {ERA, FIP, WHIP} | `rotostats_sgp_denom_inverse_ERA,FIP,WHIP` | YES |
| TC-SGP-INV-4 | {ERA, FIP} | `rotostats_sgp_denom_inverse_ERA,FIP` | YES (tester deviation — see below) |
| TC-SGP-INV-5 | {FIP, WHIP} | `rotostats_sgp_denom_inverse_FIP,WHIP` | YES |
| TC-SGP-INV-6 | {FIP} | `rotostats_sgp_denom_inverse_FIP` | YES |

**Tester pragmatic deviation — TC-SGP-INV-4:** Planner's Respawn-1 revision selected `config$inverse_categories = c("ERA")` as the novel id. However, `{ERA}` alone was already pre-registered by T-25 (line 484 of `test-sgp-denominators.R`: `sgp_denominators(h, "ERA", ...)` — Layer-3 fires, effective set `{ERA}`). Tester independently identified this residual suppression and resolved it by using `config$inverse_categories = c("ERA","FIP")` with `.lh_era_whip_fip()` fixture, yielding id `rotostats_sgp_denom_inverse_ERA,FIP` — genuinely novel. Behavioral assertion ("message says 'from config'") is preserved unchanged.

### Problems Encountered and Resolutions

| # | Problem | Signal | Routed To | Resolution |
| --- | --- | --- | --- | --- |
| 1 | TC-INV-2: `expect_equal(result, toupper(result))` self-contradicts mixed-case `"xFIP"` / `"xERA"` in the starter list | BLOCK (F1) | planner | Dropped the `toupper` assertion in Respawn-1 test-spec.md revision. `setequal` + `length` + `no-duplicates` fully verify correctness without over-constraining casing. |
| 2 | `validate_inverse_categories()` in `R/league-config.R` used `"element{?s}"` without a preceding count variable, causing cli to throw "Cannot pluralize without a quantity" | BLOCK (F2) | builder | Builder Respawn-1 (commit `00b68c8`) added `n_bad <- length(bad)` and changed to `"{n_bad} element{?s} of {.arg inverse_categories} not in {.arg categories}."` |
| 3 | TC-SGP-INV-2: `withCallingHandlers` captured no message — `cli_inform(.frequency = "once")` had already registered the id `"rotostats_sgp_denom_inverse_ERA,WHIP"` in TC-SGP-INV-1 via `suppressMessages` | BLOCK (F3) | planner | Planner Respawn-1: changed TC-SGP-INV-2's scoring categories from `c("ERA","WHIP")` to `c("WHIP")` only, producing effective set `{WHIP}` and novel id `WHIP`. |
| 4 | TC-SGP-INV-4: `withCallingHandlers` captured no message — planner's revision used `c("ERA")` as the id, but `{ERA}` was already pre-registered by T-25 in the existing test suite | BLOCK (F4, plus residual suppression discovered by tester) | planner (initial); tester (residual) | Tester independently chose `config$inverse_categories = c("ERA","FIP")` with `.lh_era_whip_fip()` fixture, yielding novel id `ERA,FIP`. |
| 5 | TC-SGP-INV-5: `withCallingHandlers` captured no message — same once-suppression issue | BLOCK (F5) | planner | Planner Respawn-1: changed to `.lh_era_whip_fip()` fixture with `scoring_categories = c("WHIP","FIP")`, yielding effective set `{FIP,WHIP}` and novel id `FIP,WHIP`. |

### Review Summary

Pending — reviewer review follows scriber.

- **Pipeline isolation**: pending
- **Convergence**: N/A (pure infrastructure run, no simulation)
- **Tolerance integrity**: pending
- **Verdict**: pending

## Design Decisions

1. **Option (a) for Layer-2 — new `config = NULL` arg on `sgp_denominators()`**: Three candidate approaches were evaluated. Option (a) — new `config` arg — was chosen because it is minimal, explicit, and consistent with the existing `convert_rate_stats(league_config = NULL)` pattern. Option (b) — attach config to `league_history` — was rejected: `league_history`'s own header comment reads "Holds only historical data. Structural league settings live in league_config()" — adding behavioral configuration to a data-holding object violates its design contract. Option (c) — punt Layer-2 — was rejected because it would violate plan acceptance criterion §5 ("inherits from config$inverse_categories when the user passes the config and leaves the arg NULL").

2. **Full-replacement override semantics (Layer-1 wins unconditionally)**: When the user passes both an explicit `inverse_categories` value and a `config` object, Layer-1 wins and config is never consulted. No augmentation or merging occurs. This matches the package's existing philosophy — explicit beats implicit — and avoids a source of hard-to-debug surprises where a user testing a specific configuration gets silently merged results.

3. **Mixed-case `xFIP`/`xERA` in `INVERSE_CATEGORIES`**: The constant stores `"xFIP"` and `"xERA"` (lowercase x prefix) rather than `"XFIP"`/`"XERA"`. This is conventional FanGraphs notation and matches how these categories appear in league history data files. A blanket `toupper()` assertion on the returned vector is therefore inappropriate and was removed from TC-INV-2 in Respawn-1. User-supplied inputs are still normalized to uppercase for membership checking inside `validate_inverse_categories()` and Layer-1 resolution.

4. **`!is.null()` detection instead of `missing()` for Layer-1**: The old `inverse_categories_is_default <- missing(inverse_categories)` flag was eliminated. With `NULL` as the new default, `!is.null(inverse_categories)` detects Layer-1 directly. The `!is.null()` form is simpler, avoids `missing()`'s subtle semantics in `do.call()` contexts, and removes one variable from the resolution block with no behavioral change.

5. **Layer-3 as intersection (not the full lookup)**: Layer-3 returns `intersect(scoring_categories, inverse_categories())` rather than the full `INVERSE_CATEGORIES` vector. This ensures that a batting-only league never gets ERA/WHIP direction-flips, and a FIP-scoring league automatically gets FIP direction-flips. The intersection formula is what the old hardcoded default was implicitly doing (`intersect(scoring_categories, c("ERA","WHIP"))`), extended to 8 entries. This is the only Layer-3 approach that preserves the P4 invariant: "effective set is always a subset of scoring_categories."

6. **Minimum-viable config consultation surface**: Only `config$inverse_categories` is read in `sgp_denominators()`. No other config fields (`config$categories`, `config$n_teams`, etc.) are consulted. This is an explicit "additive for the future" decision — Plan B (`sgp()` refactor) and future consumers (`zaa()`, `zar()`, `pvm()`) can consult additional config fields when needed, without the current run creating hidden cross-field coupling.

## Handoff Notes

### For future consumers: zaa() / zar() / pvm()

These functions do not yet exist but will consume the inverse-categories infrastructure. When building them:

1. **Accept `config = NULL`** as an argument (same as `sgp_denominators()` in this run). The three-layer resolution logic can be copied verbatim or extracted to a shared helper. The `intersect(scoring_categories, inverse_categories())` Layer-3 fallback should be used consistently.

2. **Do NOT re-validate `config$inverse_categories`** when inheriting from config (Layer-2). It is already validated and uppercased at `league_config()` construction time.

3. **The `$inverse_categories` field is NULL by default** on `league_config()` objects. Test for `!is.null(config$inverse_categories)` before using it as Layer-2.

### For Plan B: sgp() refactor

`R/sgp.R` currently hardcodes ERA/WHIP/AVG directionality in its body (not via `inverse_categories`). Plan B addresses this separately in `plans/sgp-advanced-rate-stats.md`. The infrastructure added in this run does NOT change `sgp.R` behavior — that file is frozen.

When Plan B lands, `sgp()` will be able to call `inverse_categories()` for its own resolution. At that point, the Note 6 in Key Design Decisions (sgp-2026-04-16) — "`INVERSE_CATEGORIES` constant replaced by `inverse_categories` argument" — will be fully superseded.

### cli once-suppression gotcha for test authors

`cli_inform(.frequency = "once", .frequency_id = id)` registers the id session-wide in a single R process (the testthat test session). Any subsequent call with the same id is silently suppressed — `withCallingHandlers` will capture nothing. When writing tests that capture the `cli_inform` message:

- **Never reuse the same effective inverse set across message-capture tests in the same file.** Each `withCallingHandlers`-based test must use a unique scored-category set so the effective `intersect()` result (and therefore the `.frequency_id` suffix) is unique at the point that test executes.
- Check `test-sgp-denominators.R` for the final frequency ID assignment table (captured in `audit.md §Frequency ID Uniqueness Map`).
- Tests that do NOT need to capture the message (like TC-SGP-INV-1 and TC-SGP-INV-9) can use `expect_no_error` or `suppressMessages` freely — these register the id but don't need to capture it.

### Validation context distinction

`league_config()` validates `inverse_categories` elements against `config$categories`. `sgp_denominators()` Layer-1 validates against `scoring_categories` (the effective scored set from `league_history`, which may differ from `config$categories` when inferred from `team_season` columns). These are correct and consistent — do not conflate the two validation contexts.

### Known limitations

- **Period > N cycles still hit max_iter** in `replacement_level()` — unrelated to this run but worth noting for orientation.
- **`character(0)` as explicit Layer-1 override disables all direction-flipping**: Passing `inverse_categories = character(0)` is `!is.null()` (zero-length character, not NULL), so it triggers Layer-1 with an empty set. This is intentional and documented in the `sgp_denominators()` roxygen, but it can be surprising if users expect `character(0)` to behave like `NULL`.
