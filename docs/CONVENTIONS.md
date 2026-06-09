# CONVENTIONS.md — Coding standards & workflow

This is an AthlyticZ teaching codebase. Optimize for a student reading it in six
months. Clear and conventional beats clever.

---

## 1. R style

Follow the **tidyverse style guide** (https://style.tidyverse.org). Key points:

- `snake_case` for variables and functions. No dots in names (`fantasy_pts`, not
  `fantasy.pts`).
  - **One deliberate exception: Shiny module functions are camelCase**
    (`playerExplorerUI()`, `playerExplorerServer()`). This matches the SlamStats house
    style (see §2 and §10). Everything else — data-layer readers, scoring helpers, pure
    utilities — stays `snake_case` (`load_player_box()`, `score_points()`). The split is:
    *module entry points* are camelCase; *everything they call* is snake_case.
- Spaces around operators, `<-` for assignment (not `=`), 80-ish column soft limit.
- Pipe with `|>` (base) or `%>%` — pick one per file; prefer base `|>` for new code.
- Functions are small and do one thing. If a function needs a comment to explain a
  block, that block probably wants to be its own function.
- Explicit namespacing in package-style code (`dplyr::filter`) is fine and encouraged
  in the data layer; in modules, attach with `library()` in `global.R`.

Run a linter (`lintr`) and styler before committing:
```r
styler::style_dir("R")
lintr::lint_dir("R")
```

---

## 2. Naming conventions (so files are predictable)

| Thing                | Pattern                  | Example                         |
|----------------------|--------------------------|---------------------------------|
| Module file          | `<feature>.R` (camelCase) | `R/modules/playerExplorer.R`    |
| Module UI fn         | `<feature>UI()`          | `playerExplorerUI()`            |
| Module server fn     | `<feature>Server()`      | `playerExplorerServer()`        |
| Nested/child module  | `<thing>Server()`        | `teamStatsServer()`             |
| Network ingest       | `ingest_<table>.R`       | `ingest_player_box.R`           |
| Cache reader         | `load_<table>()`         | `load_player_box()`             |
| Pure helper          | verb_noun                | `score_points()`, `rank_players()` |
| Cache file           | `<table>.parquet`        | `player_box.parquet`            |

> **Why camelCase modules and not the `mod_<x>_ui()` convention?** The wider Shiny
> community often uses `mod_player_ui()`/`mod_player_server()` (golem style). SlamStats —
> our proven reference — instead uses `playerUI()`/`playerServer()` files without a
> `mod_` prefix, and we match it so the two codebases read the same. We do keep the
> `R/modules/` subdirectory (SlamStats puts modules flat in `R/`) and use **descriptive**
> names (`playerExplorerUI`, not `playerUI`) for clarity.

---

## 3. Function documentation

Every exported/non-trivial function gets a roxygen-style header — even though this is
an app, not a package — because it teaches and it documents the contract:

```r
#' One-line summary.
#' @param x what it is
#' @return what comes back
#' @examples score_points(box, scoring_profiles$simple_points)
```

Inline comments explain **why**, not what. The "what" should be readable from the code.

---

## 4. Error handling

- Data layer reads must fail soft: if a cache file is missing, return an empty tibble
  with the right columns and log a warning — never `stop()` inside the running app.
- Network code (ingest/scripts) may fail loudly, but wrap `nba_*`/`espn_*` calls in
  `tryCatch()` with retry + backoff, since these APIs flake.
- Validate inputs at module boundaries (`stopifnot()` / `shiny::validate()` +
  `need()`), and show the user a friendly `f7Toast` rather than a red error.

---

## 5. Testing

- **Unit tests** (`tests/testthat/`) for every data-layer and scoring function. These
  are the cheapest insurance — scoring is pure, so test it hard with known stat lines.
- **Module smoke tests** with `shinytest2` for at least the critical path (search a
  player → see a value).
- A test must not hit the network. Use a small **fixture** parquet/RDS checked into
  `tests/testthat/fixtures/` (a handful of games), not live `hoopR`.
- Run before every PR: `Rscript -e 'testthat::test_dir("tests/testthat")'`.

---

## 6. Dependency management (renv)

- The project is renv-managed. `renv.lock` is the source of truth for versions and
  **is committed**.
- To set up: `renv::restore()`. To add a package: install it, confirm it's needed,
  then `renv::snapshot()` and commit the lockfile change in its own small commit.
- Never `install.packages()` into the system library and rely on it — it won't
  reproduce for the next person.

### Version alignment with SlamStats

> ⚠️ **SlamStats is not renv-managed** — there is no `renv.lock` (or `renv/`) in the
> SlamStats workspace; it just `library()`s against whatever is installed. So there is no
> lockfile to "align to." Instead, Hoop Vision's **own `renv.lock` is authoritative**,
> and it was clearly seeded from a SlamStats-style environment (it already pins the same
> charting/table stack). Below are the concrete versions currently locked (R 4.4.0).

**Shared stack — already pinned in our `renv.lock`, matching the SlamStats toolset:**

| Package         | Version   | Package         | Version  |
|-----------------|-----------|-----------------|----------|
| `shiny`         | 1.8.1.1   | `echarts4r`     | 0.4.5    |
| `shinyMobile`   | 2.0.1     | `r2d3`          | 0.2.6    |
| `dplyr`         | 1.1.4     | `reactable`     | 0.4.4    |
| `tidyr`         | 1.3.1     | `reactablefmtr` | 2.0.0    |
| `lubridate`     | 1.9.3     | `shinyjs`       | 2.1.0    |
| `purrr`         | 1.0.2     | `tidyRSS`       | 2.0.7    |
| `tibble`        | 3.2.1     | `testthat`      | 3.2.1.1  |
| `scales`        | 1.3.0     | `memoise`       | 2.0.1    |
| `RColorBrewer`  | 1.1-3     | `htmlwidgets`   | 1.6.4    |

**Flags — gaps and oddities to fix before/at the relevant phase (don't adopt blindly):**

- ❗ **`hoopR` is NOT in the lockfile.** Our core NBA data source is unlocked. Add it in
  Phase 0/1 (`install.packages("hoopR")` → confirm → `renv::snapshot()`), targeting a
  current release (~v3.1.0 per `CLAUDE.md §2`). Note: **`wehoop`** (hoopR's WNBA sibling,
  same SportsDataverse codebase) *is* pinned — it's a leftover from the SlamStats lineage,
  not needed for an NBA app. Leave it until `hoopR` lands, then remove it in a
  `chore/renv` commit; meanwhile it's a useful version reference for the sibling package.
- ❗ **`arrow` is NOT in the lockfile.** The documented parquet cache format
  (`docs/DATA_HOOPR.md §4`) needs it. Add it in Phase 1 before writing any ingest.
- ❗ **`shinytest2` is NOT in the lockfile.** Module smoke tests (§5) depend on it. Add it
  before Phase 3.
- ✅ **`apexcharter` is absent** — consistent with our decision to standardize on
  `echarts4r` + `r2d3`. Nothing to remove.

When you add any of the above: install, confirm it's needed, `renv::snapshot()`, and
commit the lockfile change in its own small `chore/renv-*` commit (per §7).

---

## 7. Git & PR workflow

- Branch per feature: `feat/player-explorer`, `fix/score-na-handling`,
  `chore/renv-bump`.
- Small, focused commits with present-tense messages: `add rolling-average helper`.
- Open a PR into `main`; keep diffs reviewable (one module/concern per PR).
- PR description checklist:
  - [ ] tests added/updated and passing
  - [ ] docs in `docs/` updated if behavior/contract changed
  - [ ] no secrets / no `data-cache/` committed
  - [ ] `renv.lock` updated if deps changed
- `main` should always run.

---

## 8. Secrets & config

- No API keys are required for the core hoopR sources, but if any are added (e.g. a
  proxy for stats.nba.com), keep them in `.Renviron` (gitignored) and read with
  `Sys.getenv()`. Never commit secrets.
- App config (default season, default scoring profile, cache paths) lives in one
  `R/utils/config.R`, not scattered as magic numbers.

---

## 9. .gitignore essentials

```
.Rproj.user
.Rhistory
.RData
.Renviron
renv/library/
data-cache/
www/cache/
*.log
```

(`renv.lock` and `renv/activate.R` ARE committed; the library is not.)

---

## 10. Learning from SlamStats

SlamStats is the proven reference implementation. When its source is available in the
workspace, before building a new feature:
1. Find the analogous SlamStats module and read how it structures UI + server.
2. Match its module contract, naming, and theming approach.
3. Reuse its helper patterns (formatting, theming, cache reads) rather than reinventing.
4. Note any deliberate differences (NBA vs the sport SlamStats covers) in the PR.

Treat SlamStats as the "house style" for anything these docs don't specify.

**What we match** (the house style): camelCase module naming (§2), the shared `r`
reactiveValues pattern, nested modules, `f7VirtualList` + client-side searchbar, JS→Shiny
click bridges, lazy chart rendering, the dark-mode reactable re-theme, `echarts4r`/`r2d3`
charts, and `reactablefmtr::color_tiles()` for color-scaled tables (see
`docs/SHINYMOBILE_PATTERNS.md`).

**Where we deliberately diverge** (SlamStats is looser here; do *not* copy it):

| Topic                    | SlamStats does…                                  | Hoop Vision does… (and why)                                   |
|--------------------------|--------------------------------------------------|--------------------------------------------------------------|
| Network access           | Calls `wehoop::espn_*` / RSS **inside modules & UI** (`player.R`, `home.R`, `live.R`) | **Quarantines all network in `R/data/ingest_*.R` + `scripts/`** — modules only read the cache (`docs/DATA_HOOPR.md`). Keeps the app offline-ish and testable. |
| Data files               | Commits a static `data/*.rds` snapshot to git    | **Gitignores `data-cache/`** and rebuilds it via `scripts/refresh_cache.R`. Generated data never enters the repo. |
| Cache format             | `readRDS()` of `.rds`                             | **Parquet via `arrow`** (columnar, filterable) per `docs/DATA_HOOPR.md §4`. |
| Load strategy            | `readRDS()` **everything** at startup            | **Preload only small tables; lazy-read big ones** (`docs/ARCHITECTURE.md §6`). |
| Module args              | Plain data frames + shared `r` only              | **Hybrid**: static frames plain, user-selected state reactive (`docs/ARCHITECTURE.md §4`). |
| Dependencies             | Not renv-managed                                 | **renv-managed**; `renv.lock` is authoritative (§6). |

Call out any further intentional difference in the PR description.
