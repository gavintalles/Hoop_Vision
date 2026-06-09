# FANTASY_SPEC.md — Fantasy scoring & feature spec

This defines *what the app computes* and *what the user sees*. Scoring is
**config-driven**: formulas live in data, not in module code.

---

## 1. Scoring profiles (config, not hardcoded)

A scoring profile is a named set of per-stat weights. Define them in one place
(`R/data/scoring_profiles.R` returning a list) so modules just pick one.

```r
scoring_profiles <- list(
  draftkings = list(
    label = "DraftKings",
    weights = c(points = 1, three_point_field_goals_made = 0.5,
                rebounds = 1.25, assists = 1.5, steals = 2, blocks = 2,
                turnovers = -0.5),
    bonuses = list(double_double = 1.5, triple_double = 3)
  ),
  fanduel = list(
    label = "FanDuel",
    weights = c(points = 1, rebounds = 1.2, assists = 1.5,
                blocks = 3, steals = 3, turnovers = -1),
    bonuses = list()
  ),
  simple_points = list(
    label = "Simple Points",
    weights = c(points = 1, rebounds = 1, assists = 1,
                steals = 1, blocks = 1, turnovers = -1),
    bonuses = list()
  )
)
```

> Verify the DraftKings/FanDuel weights against current official rules before
> publishing — DFS sites adjust them. Treat the values above as sensible defaults to
> start from, and keep them editable in one file.

**Category leagues** (H2H / roto) don't use weights; they compare raw category totals
or rates. Standard **9-CAT**: `FG%`, `FT%`, `3PM`, `PTS`, `REB`, `AST`, `STL`, `BLK`,
`TO` (TO counts against). **8-CAT** drops `TO`. Represent a category profile as the
ordered set of category columns plus a direction (higher-is-better, except TO).

The app should let the user **switch profiles** (an `f7Radio`/`f7Select` in a settings
sheet) and recompute everywhere. The active profile is the shared `scoring()` reactive
(see `docs/ARCHITECTURE.md` §5).

---

## 2. The scoring function (pure, tested)

`R/data/fantasy_score.R`:

```r
#' Compute fantasy points for player-game rows under a points profile.
#' @param box tibble of player_box rows (already cleaned)
#' @param profile one element of scoring_profiles (points type)
#' @return box with an added `fantasy_pts` column
score_points <- function(box, profile) {
  w <- profile$weights
  stat_cols <- intersect(names(w), names(box))
  base <- as.matrix(box[stat_cols]) %*% w[stat_cols]
  box$fantasy_pts <- as.numeric(base)
  box <- apply_bonuses(box, profile$bonuses)   # double/triple-double etc.
  box
}
```

Requirements:
- **Pure**: same input → same output, no I/O, no reactivity. Easy to unit-test.
- Handle `NA` stats as 0 *for scoring* (a DNP scores 0), but keep DNP flags for
  display so we don't show a fake "0-point game" as a real performance.
- Double-double / triple-double = ≥10 in 2 / 3 of {PTS, REB, AST, STL, BLK}.
- Write `testthat` tests with known stat lines (e.g. a 30/10/10 line under each
  profile) so changes to weights can't silently break scoring.

---

## 3. Aggregations the app needs

From the per-game scored table, derive:

- **Season averages** — mean `fantasy_pts` and per-stat means, regular season only by
  default (`season_type == 2`), excluding DNPs.
- **Trailing window** — rolling mean over the last *N* games (user-set via stepper,
  default 7 and 15). This is the "hot/cold" signal.
- **Per-36 / per-game** toggle — normalize counting stats by minutes when requested.
- **Consistency** — standard deviation of `fantasy_pts` (a low-variance 35-pt player is
  more valuable in H2H than a boom/bust one).
- **Rank / percentile** — position within the league for the active profile.

Keep these as pure functions in `R/data/` so both the rankings and player modules reuse
them.

---

## 4. Projections (phased — keep MVP honest)

Don't over-promise predictive power early.

- **MVP (Phase 4):** projection = blend of season-to-date average and trailing-N
  average (e.g. `0.5*season + 0.5*last15`). Simple, transparent, defensible.
- **Later:** opponent adjustment (defense-vs-position from team_box / pbp), minutes
  trend, rest/back-to-back flags from the schedule, injury status (manual or external).
- **Always** show projections as a range or with a "low confidence early in season"
  note. Label clearly that they are estimates.

---

## 5. Features & screens (maps to the bottom tabs)

| Tab        | Module                | What it does                                             |
|------------|-----------------------|----------------------------------------------------------|
| Today      | `mod_today`           | tonight's games (`espn_nba_scoreboard` cache), notable fantasy lines |
| Players    | `mod_player_explorer` | search a player → card with averages, trend chart, game log, fantasy value |
| Rankings   | `mod_rankings`        | sortable leaderboard of fantasy value (season + trailing), filter by position |
| Compare    | `mod_compare`         | pick 2–3 players → side-by-side stats + radar chart      |
| Team       | `mod_team`            | build a roster, see combined category/points outlook, simple matchup |

Cross-cutting: a **settings sheet** (`f7Sheet`) for scoring profile, season, and
season-type toggle.

---

## 6. MVP definition (what "done" means for v0.1)

A user on a phone can:
1. Open the app (cache already built) and land on a working tab layout.
2. Search any current-season NBA player and see their per-game averages + a trend chart.
3. See that player's **fantasy value** under at least the "Simple Points" profile.
4. Open a **rankings** table sorted by fantasy value.
5. Switch the scoring profile and watch values/rankings update.

Everything else (compare radar, team builder, projections, live "today" feed, PWA
install, deploy) is post-MVP and scheduled in `docs/PROJECT_PLAN.md`.

---

## 7. Honest limitations to document in-app

- Data is "as of" the last cache refresh (show the timestamp).
- Projections are simple averages, not a predictive model (early phases).
- ESPN box-score data is the source of truth here; it can differ slightly from
  stats.nba.com advanced figures.
- This is an educational tool, not betting advice.
