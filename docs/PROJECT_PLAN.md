# PROJECT_PLAN.md — Building Hoop Vision

The comprehensive, phased plan for building the NBA fantasy app. **Build in this
order.** Each phase has a goal, concrete deliverables, and a "definition of done" gate
that must pass before moving on. Claude Code should treat each phase as a milestone and
each deliverable as a candidate branch/PR.

> Phases 0–4 are the MVP. Phases 5–8 are enhancements. Don't start a later phase before
> the earlier gate passes — especially: **no projections or team builder before the
> data layer and player views work.**

---

## Phase 0 — Project setup (foundation)

**Goal:** an empty but correct, reproducible project that runs.

Deliverables
- Initialize the repo structure from `CLAUDE.md` §3 (create `R/`, `R/data/`,
  `R/modules/`, `R/utils/`, `scripts/`, `tests/testthat/`, `www/`, `docs/`).
- `renv` initialized; install base stack (shiny, shinyMobile, hoopR, dplyr, tidyr,
  lubridate, purrr, arrow, reactable, apexcharter, testthat, shinytest2); commit
  `renv.lock`.
- `.gitignore`, `.Rprofile` (renv activate), `app.R`, `global.R`, `R/ui.R`,
  `R/server.R` as thin stubs.
- A "hello world" `f7Page` with an empty `f7TabLayout` that launches.
- `R/utils/config.R` with default season, scoring profile, cache path.

**Done when:** `renv::restore()` then `shiny::runApp()` opens a blank Framework7 shell
on a phone-sized viewport without errors.

---

## Phase 1 — Data layer & cache (the backbone)

**Goal:** NBA data flows from hoopR into a local cache the app can read fast.

Deliverables
- `R/data/ingest_player_box.R`: pull `load_nba_player_box(seasons)` for the last ~3
  seasons + current, clean per `docs/DATA_HOOPR.md` §3 (coerce types, parse plus_minus,
  flag DNPs), write `data-cache/player_box.parquet`.
- `R/data/ingest_schedule.R`: `load_nba_schedule()` → `schedule.parquet`.
- `R/data/ingest_teams.R`: `espn_nba_teams()` → `teams.parquet` (ids, colors, logos).
- `R/data/ingest_player_index.R`: derive a distinct player table (id, name, position,
  current team, headshot) → `player_index.parquet`.
- `R/data/load_*.R` readers for each table (no network).
- `scripts/refresh_cache.R`: runs all ingests in order, writes `_meta.json` with
  timestamp + season range.
- Tests: a fixture-based test that `load_player_box()` returns the expected schema.

**Done when:** running `scripts/refresh_cache.R` produces the parquet files, and
`load_player_box()` returns a clean tibble with the documented columns and types.

---

## Phase 2 — Scoring engine (pure logic)

**Goal:** turn box-score rows into fantasy value, configurably.

Deliverables
- `R/data/scoring_profiles.R` (DraftKings, FanDuel, Simple Points, + 9-CAT category
  definition) per `docs/FANTASY_SPEC.md` §1.
- `R/data/fantasy_score.R`: `score_points()`, `apply_bonuses()` (double/triple-double),
  pure and NA-safe.
- `R/data/aggregate.R`: season averages, trailing-N rolling averages, per-36,
  consistency (sd), rank/percentile.
- `tests/testthat/test-scoring.R`: known stat lines (e.g. 30/10/10, a DNP, a
  double-double) scored under each profile with expected numbers.

**Done when:** scoring tests pass and `score_points()` can take the cached player_box
and return a `fantasy_pts` column for a whole season in well under a second.

---

## Phase 3 — Player explorer + Rankings (the core UX)

**Goal:** the two screens that make it a usable product.

Deliverables
- `R/modules/mod_player_explorer.R`: `f7Searchbar` → select player → `f7Card` with
  headshot, team colors, season averages, fantasy value, an `apexcharter` trend line,
  and a recent game log (`reactable`). Per-36 and trailing-N toggles.
- `R/modules/mod_rankings.R`: `reactable` leaderboard of fantasy value (season +
  trailing window), filter by position, color-scaled value column.
- `R/modules/mod_settings.R` (or a shared `f7Sheet`): pick scoring profile, season,
  season-type toggle; wired to the shared `scoring()` / `season()` reactives in
  `server.R`.
- Wire all three into `R/ui.R` + `R/server.R`.
- Smoke test: search a player → a value renders; change profile → it updates.

**Done when:** the MVP definition in `docs/FANTASY_SPEC.md` §6 is satisfied end to end
on a phone viewport.

> **MVP milestone (v0.1) reached at the end of Phase 3.** Tag it, demo it.

---

## Phase 4 — Compare, Team builder & simple projections

**Goal:** the "decision tool" features.

Deliverables
- `R/modules/mod_compare.R`: pick 2–3 players → side-by-side stat table + radar chart
  of category profile.
- `R/modules/mod_team.R`: add players to a roster (shared `roster()` reactive), show
  combined points outlook and per-category strengths/weaknesses; a simple two-roster
  matchup view.
- `R/data/projections.R`: MVP projection = blend of season + trailing-N average, shown
  with a "low confidence early season" note.
- Tests for projection math and roster aggregation.

**Done when:** a user can compare players, build a roster, and see a transparent
projected outlook, with projections clearly labeled as estimates.

---

## Phase 5 — "Today" feed & freshness

**Goal:** make it feel live without breaking the offline-cache rule.

Deliverables
- `R/data/ingest_scoreboard.R`: cache `espn_nba_scoreboard()` on a short refresh
  cadence (e.g. every 15 min during game hours) into `scoreboard.parquet`.
- `R/modules/mod_today.R`: tonight's games, notable fantasy performances, quick links
  into player explorer.
- "Data as of <timestamp>" indicator in the app footer (read `_meta.json`).
- Graceful empty states when no games / stale cache.

**Done when:** the Today tab shows current games from cache and never blocks startup or
crashes when the feed is empty/stale.

---

## Phase 6 — Polish, theming & PWA

**Goal:** make it feel like a real installable app.

Deliverables
- Finalize theme in `R/utils/theme.R` + `www/custom.css` (AthlyticZ accent, team
  colors, dark mode verified).
- Loading states, skeletons, empty states, and toasts across modules.
- PWA assets via `charpente::set_pwa()`; verify "Add to Home Screen" works and launches
  fullscreen.
- Accessibility pass: tap target sizes, contrast, font sizes.

**Done when:** the app is installable as a PWA, looks consistent in light/dark, and has
no jarring empty/loading gaps.

---

## Phase 7 — Hardening & automation

**Goal:** reliable, reproducible, maintainable.

Deliverables
- Scheduled cache refresh: a **GitHub Action** (or cron) running
  `scripts/refresh_cache.R` nightly and committing/publishing the cache to the deploy
  target (not to the repo).
- Retry/backoff + logging around all network ingests.
- CI: GitHub Action that runs `renv::restore()` + `testthat` on PRs.
- Performance pass: memoize hot rollups, debounce search, lazy-read big tables.

**Done when:** PRs are gated by passing CI, and the cache refreshes on a schedule
without manual steps.

---

## Phase 8 — Deploy

**Goal:** people can use it.

Deliverables
- Choose target (shinyapps.io → simplest; Posit Connect; or Docker per
  `docs/ARCHITECTURE.md` §7).
- Document the deploy + refresh procedure in `README.md`.
- Stage → verify on a real phone → release.

**Done when:** a public/shared URL serves the app on a phone, with data refreshing on
schedule.

---

## Cross-phase backlog (pick up opportunistically)

- Opponent-adjusted projections (defense-vs-position).
- Back-to-back / rest flags from the schedule.
- Injury status (manual flag or external source) feeding projections.
- Saved rosters / favorites (needs lightweight per-user persistence).
- Advanced stats from `nba_leaguedashplayerstats()` behind the id crosswalk
  (`docs/DATA_HOOPR.md` §5).
- Shot charts from pbp coordinates (`load_nba_pbp`) — visually great, data-heavy.

---

## How Claude Code should approach each task

1. Read `CLAUDE.md` + the relevant `docs/` file for the phase.
2. Restate the goal, list files to create/modify, name the branch.
3. Implement the smallest slice that satisfies the deliverable's "done" criteria.
4. Add tests; run them.
5. Update any doc whose contract changed.
6. Open a focused PR; check it against the PR checklist in `docs/CONVENTIONS.md`.
7. Leave `# TODO(claude): ...` notes for any deferred or uncertain decision instead of
   guessing.
