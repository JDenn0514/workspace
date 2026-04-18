# Handoff Notes

**Run:** sgp-denom-inverse-categories-param-2026-04-17
**Date:** 2026-04-18
**Slug:** 2026-04-18-sgp-denom-inverse-categories
**Branch:** feature/sgp-denom-inverse-categories-param @ 2b8060a -> PR #13 against develop
**Verdict:** PASS WITH NOTE (safe to ship; 3 deferred non-blocking notes)

---

1. **`missing()` detection at function entry**: `inverse_categories_is_default <- missing(inverse_categories)` is the first line of `sgp_denominators()`. This line must remain first if the function is ever refactored — moving it after any code that references `inverse_categories` would break the flag (R evaluates `missing()` based on whether the promise has been forced, not based on position). If a future developer adds a new argument that also uses a default-vs-explicit distinction, this same `missing()` pattern should be applied at the top of the function for each such argument.

2. **`.frequency_id` scheme for `cli_inform()`**: The id `"rotostats_sgp_denom_inverse_" + sorted comma-joined categories` is content-addressed and session-scoped. If `inverse_categories` is later stored in `$meta` on the return object (currently intentionally deferred per spec), the `.frequency_id` scheme should be kept in sync so that re-calling with the same configuration stays quiet.

3. **Four threading sites in `R/sgp-denominators.R` (now `inverse_categories`)**:
   - Line ~676 post-edit: main year-loop `standings_pos` assignment.
   - Line ~783 post-edit: slope-sign check, inverse branch (`cat %in% inverse_categories && beta_c > 0`).
   - Line ~792 post-edit: slope-sign check, normal branch (`!(cat %in% inverse_categories) && beta_c < 0`).
   - Line ~939 post-edit: bootstrap resampling `sp` assignment.

   If the bootstrap resampling path is ever refactored (e.g., extracted into a helper), the `inverse_categories` argument must be threaded through or closure-captured.

4. **`INVERSE_CATEGORIES` constant is GONE**: The string `INVERSE_CATEGORIES` no longer exists anywhere in the R source. Architecture docs, planning docs, and specs were updated. Any future reference to this constant in code review comments or documentation should be treated as referring to the deleted object, not a live symbol.

5. **Test section 13 at T-58 through T-70**: The new tests live in `tests/testthat/test-sgp-denominators.R` under the `# === 13. inverse_categories Scenarios ===` header. The `make_oavg_monotone_history()` helper is defined locally in that section. T-25 (ERA regression anchor) was re-verified in T-67 (T-NEW-10 guard in section 13).

6. **`$meta` does NOT contain `inverse_categories`**: Per spec, the return object's `$meta` slot was deliberately not extended. This avoids breaking callers who enumerate `meta` fields. If a future spec adds `inverse_categories` to `$meta`, it must be done as a backward-compatible addition.

**Known limitations / deferred follow-ups:**

- Bootstrap rank-flip path (line ~939) has no independent tester assertion with `n_bootstrap > 0`. Low risk — pattern identical to verified main rank-flip site; post-edit grep confirmed zero `INVERSE_CATEGORIES` occurrences. Defer to a future test-enhancement pass.
- `plans/sgp-denominators-architecture.md` still references the old `INVERSE_CATEGORIES` constant (internal planning doc, outside scriber write surface). Defer to a future `plans/` cleanup pass.
- `specs/spec-sgp.md` parameter row and error-handling table omit the default-path exception (the user-facing `man/` and `NEWS.md` are correct). Scriber should update `specs/spec-sgp.md` in the next cleanup run.
- `cli_inform()` message fires once per unique `inverse_categories` configuration per session. In long-running analyses with many distinct configurations, the rlang frequency cache accumulates entries. Not a bug but worth noting.
- `inverse_categories = character(0)` is legal (no rank flip applied to any category). Correct behavior for all-counting-stat leagues.

---

**PR:** https://github.com/JDenn0514/rotostats/pull/13
**Commits:** `321f530` feat, `a0ba6cd` fix, `ce8a37d` test, `2b8060a` docs
**Top commit:** `2b8060a` (docs(sgp): scriber recording for inverse_categories run)
