# CONVENTIONS.md — Coding standards & workflow

This is an AthlyticZ teaching codebase. Optimize for a student reading it in six
months. Clear and conventional beats clever.

---

## 1. R style

Follow the **tidyverse style guide** (https://style.tidyverse.org). Key points:

- `snake_case` for variables and functions. No dots in names (`fantasy_pts`, not
  `fantasy.pts`).
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
| Module file          | `mod_<feature>.R`        | `mod_player_explorer.R`         |
| Module UI fn         | `mod_<feature>_ui()`     | `mod_player_explorer_ui()`      |
| Module server fn     | `mod_<feature>_server()` | `mod_player_explorer_server()`  |
| Network ingest       | `ingest_<table>.R`       | `ingest_player_box.R`           |
| Cache reader         | `load_<table>()`         | `load_player_box()`             |
| Pure helper          | verb_noun                | `score_points()`, `rank_players()` |
| Cache file           | `<table>.parquet`        | `player_box.parquet`            |

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

> The base library set should match the SlamStats `renv.lock` where the two apps
> overlap (shiny, shinyMobile, hoopR, dplyr, apexcharter, reactable, arrow). When the
> SlamStats lockfile is available in the workspace, align versions to it before adding
> anything new.

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
