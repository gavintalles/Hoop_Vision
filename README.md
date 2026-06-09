# Hoop Vision 🏀

A mobile-first **fantasy basketball app for the NBA**, built with **R / Shiny** and
**`shinyMobile`**, powered by NBA data from **`hoopR`** (SportsDataverse).

Hoop Vision is the NBA companion to **SlamStats**. It is an **AthlyticZ** educational
project — the codebase is meant to be read and learned from, not just run.

---

## Quick start

```bash
# 1. Clone
git clone https://github.com/gavintalles/Hoop_Vision.git
cd Hoop_Vision

# 2. Install R (>= 4.3) and restore the locked dependencies
Rscript -e 'install.packages("renv")'
Rscript -e 'renv::restore()'

# 3. Build the local data cache (first run pulls a few NBA seasons)
Rscript scripts/refresh_cache.R

# 4. Run it
Rscript -e 'shiny::runApp(".", launch.browser = TRUE)'
```

The app opens as a Framework7 mobile interface. For the real feel, open your browser
dev tools and switch to a phone viewport, or load it on your phone over your local
network.

---

## What's inside

| Feature              | Description                                                       |
|----------------------|-------------------------------------------------------------------|
| Player explorer      | Search any NBA player, see stat splits and trends                 |
| Fantasy value        | Per-game fantasy points under configurable scoring profiles       |
| Player compare       | Head-to-head comparison of two or more players                    |
| Rankings             | Sortable fantasy leaderboards (season + trailing window)          |
| Matchup / team       | Build a roster and project category or points outcomes            |

See `docs/PROJECT_PLAN.md` for the full phased roadmap and `docs/FANTASY_SPEC.md`
for scoring details.

---

## How the project is documented

This repo is set up so that an LLM coding assistant (Claude Code) and a human
developer share the same context:

- **`CLAUDE.md`** — start here; the operating rules for the codebase.
- **`docs/PROJECT_PLAN.md`** — phased build plan and milestones.
- **`docs/ARCHITECTURE.md`** — how the layers fit together and the data flow.
- **`docs/DATA_HOOPR.md`** — the hoopR data layer, fields, and caching rules.
- **`docs/SHINYMOBILE_PATTERNS.md`** — UI framework conventions.
- **`docs/CONVENTIONS.md`** — coding style, git, testing, and renv workflow.
- **`docs/FANTASY_SPEC.md`** — fantasy scoring models and feature scope.

---

## Data & attribution

Basketball data is accessed via [`hoopR`](https://hoopr.sportsdataverse.org), which
sources from ESPN and the NBA Stats API. Respect those services: the app caches data
locally and refreshes on a schedule rather than hammering the APIs. See the data-layer
rules in `CLAUDE.md` §4.

## Contributors

- **gavintalles** — repository owner
- **AthlyticZ** — project sponsor / contributor

## License

TBD — add a `LICENSE` file before any public release.
