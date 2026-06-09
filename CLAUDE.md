# CLAUDE.md — Hoop Vision

> This file is read automatically by Claude Code at the start of every session.
> It is the **single source of truth** for how to work in this repo. Read it fully
> before writing any code. The deeper references live in `docs/` — read the
> relevant one before starting a feature, not after.

---

## 1. What we are building

**Hoop Vision** is an interactive, **mobile-first fantasy basketball app for the NBA**,
built in **R / Shiny** using the **`shinyMobile`** framework (Framework7 under the hood).
All basketball data comes from the **`hoopR`** package (SportsDataverse).

It is the NBA sibling of an existing production app, **SlamStats**, which was also
built with `shinyMobile`. Where a decision is ambiguous, prefer the pattern SlamStats
already uses (see `docs/CONVENTIONS.md` for how to surface those patterns when the
SlamStats source is available in the workspace).

This is an educational project run by **AthlyticZ** with a student as the primary
developer. Code should be **readable and teachable first**, clever second. Favor
clear, well-commented modules over dense one-liners.

---

## 2. Tech stack (pin exact versions from `renv.lock`)

| Layer            | Tool                                              |
|------------------|---------------------------------------------------|
| Language         | R (>= 4.3)                                         |
| Reactive runtime | `shiny`                                            |
| UI framework     | `shinyMobile` (Framework7; ~v2.0.1)               |
| Data source      | `hoopR` (SportsDataverse; ~v3.1.0)                |
| Data wrangling   | `dplyr`, `tidyr`, `lubridate`, `purrr`            |
| Tables           | `reactable` (or `f7Table` for simple cases)       |
| Charts           | `apexcharter` (pairs natively with Framework7)    |
| Local cache      | `arrow` (parquet) and/or `duckdb`; `qs`/`readr`   |
| Dependency mgmt  | `renv`                                             |
| Tests            | `testthat`, `shinytest2`                           |

> **Do not invent versions.** When the project's `renv.lock` is present, treat it as
> authoritative and match it. If a needed package is not in the lockfile, propose
> adding it (and run `renv::snapshot()`) rather than silently `install.packages()`.

---

## 3. Repository map

```
Hoop_Vision/
├── CLAUDE.md                 # you are here
├── README.md                 # human onboarding
├── renv.lock                 # locked dependencies (source of truth for versions)
├── .Rprofile                 # activates renv
├── app.R                     # thin entrypoint: source global, ui, server
├── global.R                  # libraries, options, load cached data, source modules
├── R/                        # all application code
│   ├── ui.R                  # f7Page + f7TabLayout assembly
│   ├── server.R              # top-level server, wires modules together
│   ├── data/                 # the DATA LAYER (see docs/DATA_HOOPR.md)
│   │   ├── ingest_*.R        #   functions that CALL hoopR and write the cache
│   │   ├── load_*.R          #   functions the app calls to READ the cache
│   │   └── fantasy_score.R   #   pure scoring functions (see docs/FANTASY_SPEC.md)
│   ├── modules/              # one Shiny module per feature (mod_*.R)
│   └── utils/                # helpers, theming, formatting
├── data-cache/               # generated parquet/duckdb (gitignored)
├── scripts/                  # one-off + scheduled refresh scripts
│   └── refresh_cache.R       # nightly data pull (cron / GitHub Action)
├── tests/
│   └── testthat/
├── www/                      # static assets, PWA manifest, icons, custom css
└── docs/                     # the "training" docs — read these
    ├── PROJECT_PLAN.md       # phased build plan + milestones
    ├── ARCHITECTURE.md       # layers, data flow, module contract
    ├── DATA_HOOPR.md         # hoopR reference + data dictionary + caching rules
    ├── SHINYMOBILE_PATTERNS.md  # UI framework patterns and gotchas
    ├── CONVENTIONS.md        # coding standards, git, testing, renv workflow
    └── FANTASY_SPEC.md       # scoring models + feature spec + MVP scope
```

---

## 4. The data layer is the most important rule

Read `docs/DATA_HOOPR.md` before touching any `hoopR` call. The non-negotiables:

1. **Never call `hoopR` directly from a Shiny module or `server` function.**
   Modules read from the **local cache** via `R/data/load_*.R`. Only the ingest
   scripts in `R/data/ingest_*.R` and `scripts/refresh_cache.R` are allowed to hit
   the network.

2. **Use the right `hoopR` family for the job:**
   - `load_nba_*()` → bulk/historical seasons. Fast, pre-built, safe to call often.
   - `espn_nba_*()` → live / current game data, rosters, schedules. Stable.
   - `nba_*()` → deep box scores, advanced & tracking stats from stats.nba.com.
     **Rate-limited and fragile.** Always cache, always add delays, never loop
     hundreds of calls without throttling.

3. **The app must work offline-ish.** If the cache is missing or a refresh fails,
   the UI degrades gracefully (show "data unavailable", never crash). Never block
   app startup on a live network call.

4. **IDs from ESPN and stats.nba.com are different namespaces. Never join across
   them on raw id.** Use the crosswalk approach described in `docs/DATA_HOOPR.md`.

---

## 5. UI rules (see `docs/SHINYMOBILE_PATTERNS.md`)

- The app is **mobile-first**. Design for a ~390px-wide phone screen; desktop is a
  bonus, not the target.
- Top-level layout is **`f7TabLayout`** with a small number of bottom tabs.
- Every feature is a **Shiny module** rendered inside a tab. No business logic in
  `ui.R`.
- Prefer native Framework7 components (`f7Card`, `f7List`, `f7Searchbar`, `f7Sheet`,
  `f7Toast`, `apexcharter`) over generic Shiny widgets so the app feels native.
- Theme with team / AthlyticZ colors via the `f7Page` options + a single CSS file in
  `www/`. Do not scatter inline styles across modules.

---

## 6. How to work in this repo (workflow)

1. **Plan before coding.** For any non-trivial task, restate the goal, list the files
   you will touch, and confirm the approach against `docs/PROJECT_PLAN.md`. Build in
   the phase order defined there — do not jump ahead to projections before the data
   layer and player views exist.
2. **Small, reviewable changes.** One feature/module per branch and PR. Keep diffs
   focused. See branching + PR rules in `docs/CONVENTIONS.md`.
3. **Write the function, then the test.** New data-layer or scoring functions get a
   `testthat` test. New modules get at least a smoke test.
4. **Comment for a student reader.** Explain *why*, especially around any `hoopR`
   quirk or fantasy-scoring decision.
5. **Update docs when behavior changes.** If you change the cache schema, scoring
   defaults, or module contract, update the matching file in `docs/`.

---

## 7. Common commands

```bash
# Restore the exact dependency set
Rscript -e 'renv::restore()'

# Refresh the local data cache (run before first launch, then nightly)
Rscript scripts/refresh_cache.R

# Run the app locally
Rscript -e 'shiny::runApp(".", launch.browser = TRUE)'

# Run tests
Rscript -e 'testthat::test_dir("tests/testthat")'

# After adding/removing a package
Rscript -e 'renv::snapshot()'
```

---

## 8. Guardrails — do NOT

- ❌ Call `nba_*()` stats.nba.com endpoints in a tight loop without throttling.
- ❌ Commit `data-cache/`, secrets, or `.Renviron` (they are gitignored).
- ❌ Hardcode a scoring formula inside a module — scoring is config-driven
  (`docs/FANTASY_SPEC.md`).
- ❌ Mix ESPN ids and stats.nba.com ids in a join.
- ❌ Add a dependency without updating `renv.lock`.
- ❌ Put data-fetching logic inside reactive/UI code.
- ❌ Ship a feature with no test and no doc update.

When unsure, prefer the conservative choice and leave a `# TODO(claude): ...` note
explaining the open question rather than guessing silently.
