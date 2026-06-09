# ARCHITECTURE.md — How Hoop Vision fits together

A small, layered Shiny app. The whole point of the architecture is to keep
**network access**, **business logic**, and **UI** in separate places so each can be
tested and reasoned about on its own.

---

## 1. The four layers

```
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER         f7Page + f7TabLayout, one module per tab    │
│                   (R/ui.R, R/modules/mod_*.R UI functions)    │
├─────────────────────────────────────────────────────────────┤
│  MODULE/SERVER    reactive logic per feature; reads cache,    │
│                   applies scoring, renders tables & charts    │
│                   (R/modules/mod_*.R server functions)        │
├─────────────────────────────────────────────────────────────┤
│  SERVICE LAYER    pure functions: scoring, projections,       │
│                   formatting, filtering (R/data/, R/utils/)   │
├─────────────────────────────────────────────────────────────┤
│  DATA LAYER       read from local cache (R/data/load_*.R)     │
│                   ◄── cache built offline by ingest_*.R ──►    │
└─────────────────────────────────────────────────────────────┘
        ▲ network access lives ONLY here (and in scripts/)
```

The golden rule (repeated from `CLAUDE.md` because it matters most): **network calls
happen only in `R/data/ingest_*.R` and `scripts/refresh_cache.R`.** Everything above
the data layer reads from `data-cache/`.

---

## 2. Runtime data flow

```
  scheduled (nightly)                 at app runtime
  ───────────────────                 ──────────────
  refresh_cache.R                     user opens a tab
        │                                   │
        ▼                                   ▼
  hoopR::load_nba_* ()              mod_*  server function
        │                                   │
        ▼                                   ▼
  ingest_*.R  (clean,               load_*.R  (read parquet,
   coerce, attach scoring)           filtered)
        │                                   │
        ▼                                   ▼
  data-cache/*.parquet  ───────────►  fantasy_score() / projection helpers
                                            │
                                            ▼
                                      reactable / apexcharter  →  f7Card
```

`global.R` loads the small, always-needed cache tables (player index, team directory,
current schedule) **once at startup** into memory. Large tables (full player_box) are
read on demand inside modules with column/row filters, not held entirely in memory.

---

## 3. Files and responsibilities

| File / dir                | Responsibility                                                  |
|---------------------------|-----------------------------------------------------------------|
| `app.R`                   | 3 lines: `source("global.R")`, build `ui`, define `server`, run |
| `global.R`                | libraries, `options()`, source all `R/**`, preload small caches |
| `R/ui.R`                  | assemble `f7Page(f7TabLayout(...))` from module UIs             |
| `R/server.R`              | call each module's server; share top-level reactives            |
| `R/data/ingest_*.R`       | **network**: call hoopR, clean, write parquet                   |
| `R/data/load_*.R`         | **no network**: read parquet, return tibble                     |
| `R/data/fantasy_score.R`  | pure scoring functions (see `docs/FANTASY_SPEC.md`)             |
| `R/data/projections.R`    | rolling-average / rest-of-season projections                    |
| `R/modules/mod_*.R`       | one feature each: `mod_<x>_ui()` + `mod_<x>_server()`           |
| `R/utils/theme.R`         | colors, team palette, f7 theme options                          |
| `R/utils/format.R`        | number/stat formatting helpers                                  |
| `scripts/refresh_cache.R` | orchestrates all ingest_*.R; entrypoint for cron/Action         |
| `www/`                    | custom.css, PWA manifest, service worker, icons                 |

---

## 4. The Shiny module contract

Every feature is a module. Follow this shape exactly so modules stay swappable and
testable:

```r
# R/modules/mod_player_explorer.R

#' Player Explorer UI
#' @param id module id
mod_player_explorer_ui <- function(id) {
  ns <- NS(id)
  f7Tab(
    tabName = "Players", icon = f7Icon("person_2_fill"),
    f7Searchbar(id = ns("search"), inline = TRUE),
    uiOutput(ns("player_card")),
    apexchartOutput(ns("trend"))
  )
}

#' Player Explorer server
#' @param id module id
#' @param player_box reactive returning the cached player_box tibble
#' @param scoring   reactive returning the active scoring profile
mod_player_explorer_server <- function(id, player_box, scoring) {
  moduleServer(id, function(input, output, session) {
    # 1. read inputs  2. filter cached data  3. apply service-layer functions
    # 4. render. No hoopR calls here.
  })
}
```

Rules:
- Modules receive cached data and shared state **as reactive arguments**, not by
  reaching into globals. This makes them unit-testable.
- A module never calls `hoopR`. If it needs data not in the cache, that's a signal to
  add an ingest step + cache table first.
- UI function returns an `f7Tab` (for tab-level modules) or a UI fragment.

---

## 5. Shared state

Keep top-level shared reactives in `server.R` and pass them down:

- `scoring()` — the active scoring profile (default + user-selected). Lives at the top
  so every module scores consistently. See `docs/FANTASY_SPEC.md`.
- `season()` — selected season, defaulting to `most_recent_nba_season()`.
- `roster()` — the user's built team (for the matchup/team module).

Avoid deep reactive chains. If two modules need to talk, route it through a small
shared reactiveValues object created in `server.R`, not through the global env.

---

## 6. Caching & performance notes

- Preload only small tables at startup; lazy-read big ones.
- Memoize expensive pure computations within a session (e.g. `memoise::memoise()` on a
  scoring rollup keyed by `(season, scoring_profile)`).
- `reactable` handles large tables client-side well; avoid re-rendering on every
  keystroke — debounce search inputs.
- The cache refresh is the slow part and it runs **offline**, so the live app stays
  snappy.

---

## 7. Deployment (later phase)

Target options, in rough order of simplicity:
1. **shinyapps.io** — easiest; bundle the cache or refresh on a schedule externally.
2. **Posit Connect** — if AthlyticZ has it; supports scheduled refresh + the app.
3. **Docker + any host** — most control; `rocker/shiny` base image + renv restore.

PWA assets (`www/manifest.webmanifest`, service worker) let users "install" the app on
their phone home screen — set `allowPWA = TRUE` in `f7Page` and generate assets with
`charpente::set_pwa()`. See `docs/SHINYMOBILE_PATTERNS.md`.
