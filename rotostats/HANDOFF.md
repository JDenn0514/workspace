# Handoff Notes

**Run:** sgp-input-hardening-2026-04-17
**Date:** 2026-04-17
**Slug:** 2026-04-17-sgp-input-hardening
**Branch:** develop @ 240f683 (all commits landed on origin/develop)
**Verdict:** PASS_WITH_NOTE

---

1. **`withCallingHandlers` migration required for zero-playing-time callers**: Any downstream
   code that uses `withCallingHandlers(rotostats_warning_missing_category_column = ...)` to
   intercept zero-playing-time rows (0/NA IP for ERA/WHIP or 0/NA AB for AVG) must be updated
   to `withCallingHandlers(rotostats_warning_zero_playing_time = ...)`. The `NEWS.md` breaking
   changes section documents this with the exact call pattern. This is a **minor breaking
   change** for callers using `withCallingHandlers` — users catching only the error hierarchy
   (via `tryCatch`) are unaffected.

2. **Downstream stubs are now dead code**: The Step 6b code path that previously reached
   `rotostats_error_missing_config_field` for `pool_baseline = "per_player"` or
   `"universal_constants"` is now unreachable (Step 1b gates those values first). When
   implementing support for additional `pool_baseline` values, expand the `valid_pool_baselines`
   vector in Step 1b — that is the single authoritative gate. The downstream stubs can then be
   retained or removed at that time.

3. **`man/sgp.Rd` is now fully regenerated and consistent with `R/sgp.R`**: The Rd file was
   stale after the out-of-band PR #5 merge. This scriber run regenerated it via
   `devtools::document()`. Future roxygen edits to `R/sgp.R` must be followed by
   `devtools::document()` before committing. Per the r-package profile, scriber owns this step.

4. **The 5 pre-existing NOTEs are not regressions**: (a) `.playwright-mcp` hidden directory,
   (b) `unable to verify current time`, (c) non-standard top-level files, (d) NEWS.md no
   entries found, (e) Escaped LaTeX specials in replacement Rd files. These are all
   infrastructure-level issues unrelated to `sgp()`.

5. **Follow-up tasks**: See `plans/sgp-cleanup.md` for the R2/R3 follow-on tasks created in
   the out-of-band `ce1fbde` commit.

---

**Workflow violations on record (accepted by user):**

- PR #5 (`docs/cleanup-plans`) merged builder+tester commits ahead of schedule before scriber
  ran `devtools::document()`. Recovery was clean (feature/sgp-input-hardening-docs branch).
- Scriber commit `240f683` pushed directly to `origin/develop` rather than via a feature branch
  and PR. Content was correct (doc regeneration only); user chose "Accept state."
