# ARCHITECTURE.md — How Hoop Vision fits together

A small, layered Shiny app. The whole point of the architecture is to keep
**network access**, **business logic**, and **UI** in separate places so each can be
tested and reasoned about on its own.

---

## 1. The four layers

```
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER         f7Page + f7TabLayout, one module per tab    │
│                   (R/ui.R, R/modules/*.R  UI functions)       │
├─────────────────────────────────────────────────────────────┤
│  MODULE/SERVER    reactive logic per feature; reads cache,    │
│                   applies scoring, renders tables & charts    │
│                   (R/modules/*.R  server functions)           │
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
  hoopR::load_nba_* ()              <feature>Server  function
        │                                   │
        ▼                                   ▼
  ingest_*.R  (clean,               load_*.R  (read parquet,
   coerce, attach scoring)           filtered)
        │                                   │
        ▼                                   ▼
  data-cache/*.parquet  ───────────►  fantasy_score() / projection helpers
                                            │
                                            ▼
                                      reactable / echarts4r  →  f7Card
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
| `R/modules/*.R`           | one feature each: `<feature>UI()` + `<feature>Server()`          |
| `R/utils/theme.R`         | colors, team palette, f7 theme options                          |
| `R/utils/format.R`        | number/stat formatting helpers                                  |
| `scripts/refresh_cache.R` | orchestrates all ingest_*.R; entrypoint for cron/Action         |
| `www/`                    | custom.css, PWA manifest, service worker, icons                 |

---

## 4. The Shiny module contract

Every feature is a module. Follow this shape exactly so modules stay swappable and
testable. **Naming follows the SlamStats house style** (see `docs/CONVENTIONS.md` §2):
one file per feature in `R/modules/`, named in camelCase after the feature
(`playerExplorer.R`), exporting `<feature>UI()` and `<feature>Server()`. We keep
**descriptive** names (`playerExplorerUI`, not the terse `playerUI`).

```r
# R/modules/playerExplorer.R

#' Player Explorer UI
#' @param id module id
playerExplorerUI <- function(id) {
  ns <- NS(id)
  f7Tab(
    title = "Players", tabName = "players", icon = f7Icon("person_2_fill"),
    f7Searchbar(id = ns("search"), inline = TRUE),
    uiOutput(ns("player_card")),
    echarts4rOutput(ns("trend"))
  )
}

#' Player Explorer server
#' @param id module id
#' @param player_box reactive: the cached, scored player_box tibble (changes with scoring/season)
#' @param scoring    reactive: the active scoring profile
#' @param r          shared reactiveValues for cross-module signals (see §5)
playerExplorerServer <- function(id, player_box, scoring, r) {
  moduleServer(id, function(input, output, session) {
    # 1. read inputs  2. filter cached data  3. apply service-layer functions
    # 4. render. No hoopR calls here.
  })
}
```

Rules:
- A module never calls `hoopR`. If it needs data not in the cache, that's a signal to
  add an ingest step + cache table first. (SlamStats relaxes this — it calls
  `wehoop::espn_*` inside modules — but Hoop Vision keeps the quarantine; see
  `docs/DATA_HOOPR.md`.)
- UI function returns an `f7Tab` (for tab-level modules) or a UI fragment.
- Modules may **nest**: a parent server can call a child `<thing>Server()` per item,
  namespacing with `ns(<id>)`. SlamStats does this — `teamServer` spawns a
  `teamStatsServer` per team. Use it when a tab is a list of repeated sub-cards.

### How modules receive data — the hybrid argument convention

SlamStats passes everything as **plain preloaded data frames** plus one shared `r`.
That is simple but means a module can't react to the user changing the scoring profile
or season at the argument boundary. Hoop Vision uses a **hybrid** so settings changes
propagate, while still keeping the cheap things cheap:

| Argument kind                              | Passed as            | Why                                                |
|--------------------------------------------|----------------------|----------------------------------------------------|
| Static reference tables (team directory, player index) | **plain data frame** | Loaded once in `global.R`; never change at runtime — no need to wrap in `reactive()`. Matches SlamStats. |
| User-selected state (`scoring()`, `season()`) | **`reactive()`**     | The whole point is that changing them re-renders every module consistently. |
| The scored / filtered `player_box`          | **`reactive()`**     | It is derived from `scoring()`/`season()`, so it must be reactive too. |
| Cross-module signals (clicked card, "go home", re-render counter) | **shared `r` reactiveValues** | One object created in `server.R`, threaded into every module (see §5). |

Rule of thumb: **if it can change while the app is open, pass a `reactive()`; if it is
fixed at startup, pass the plain data frame.** This keeps modules unit-testable (you can
feed a `reactive()` or a static frame in a test) without the all-reactive ceremony.

---

## 5. Shared state

Keep top-level shared reactives in `server.R` and pass them down:

- `scoring()` — the active scoring profile (default + user-selected). Lives at the top
  so every module scores consistently. See `docs/FANTASY_SPEC.md`.
- `season()` — selected season, defaulting to `most_recent_nba_season()`.
- `roster()` — the user's built team (for the matchup/team module).

Avoid deep reactive chains. If two modules need to talk, route it through a small
shared reactiveValues object created in `server.R`, not through the global env.

**The shared `r` object (SlamStats pattern).** SlamStats creates one
`r <- reactiveValues(...)` in `server()` and passes it as the last argument to every
module. Hoop Vision follows this. It carries cross-cutting signals that don't belong to
any single module, for example:

```r
# in server.R
r <- reactiveValues(
  clicked_player = NULL,   # set by a JS click bridge (see SHINYMOBILE_PATTERNS §)
  chosen_tab     = NULL,    # the active tab, for tab-aware modules
  render_table   = 0        # bump to force reactable re-render after a theme change
)
```

The `render_table` counter is a real SlamStats trick: when dark mode flips we change the
global `reactable.theme` and increment `r$render_table`; every `renderReactable` does
`req(r$render_table)` so already-rendered tables redraw with the new theme. Document any
new signal you add to `r` here so it doesn't become a junk drawer.

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
