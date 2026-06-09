# SHINYMOBILE_PATTERNS.md — UI framework patterns

`shinyMobile` (RinteRface, ~v2.0.1) wraps **Framework7** to make Shiny apps that look
and feel native on iOS/Android. AthlyticZ is a funder of the package, so we are
building on home turf — lean into its components instead of fighting them with raw
HTML.

> Confirm exact function signatures against the installed version's help
> (`?f7Page`, `?f7Card`, etc.) since the API evolves. Patterns below are the stable
> core.

---

## 1. App skeleton

This mirrors the real SlamStats `app.R` skeleton.

```r
# R/ui.R

# App configuration — one list, passed to f7Page(options = ...)
app_config <- list(
  theme = "auto",                        # "ios" | "md" | "auto"
  dark = TRUE,
  color = "#E2552C",                     # AthlyticZ / Hoop Vision accent (hex OK;
                                         #   getF7Colors() names also work)
  navbar = list(mdCenterTitle = TRUE),
  preloader = FALSE,
  skeletonsOnLoad = FALSE
)

ui <- f7Page(
  title = "Hoop Vision",
  options = app_config,
  allowPWA = TRUE,                       # enables installable PWA
  f7TabLayout(
    # custom JS bridges + css (served from www/) — see the JS-bridge section below
    tags$script(src = "clickEvents.js"),
    tags$script(src = "customMessage.js"),
    tags$link(rel = "stylesheet", href = "custom.css"),
    shinyjs::useShinyjs(),
    navbar = f7Navbar(
      title = "Hoop Vision",
      rightPanel = tagList(
        tags$a(class = "link icon-only panel-open", `data-panel` = "right",
               f7Icon("gear_alt"))     # opens the Preferences panel (theme/dark/color)
      )
    ),
    panels = tagList(
      f7Panel(title = "Preferences", side = "right", effect = "push",
              # f7Radio for theme / dark / color — see Theming section
              uiOutput("prefs_panel"))
    ),
    f7Tabs(
      id = "tabs", animated = TRUE,
      todayUI("today"),
      playerExplorerUI("players", player_index),  # static ref table passed in (plain df)
      rankingsUI("rankings"),
      compareUI("compare"),
      teamUI("team")
    )
  )
)
```

- **`f7Page`** is the root. `allowPWA = TRUE` wires up the PWA path.
- **`f7TabLayout`** + **`f7Tabs`** gives the bottom tab bar — our primary navigation. The
  `f7Tabs` `id` (here `"tabs"`) matters: panel/menu links target a tab as
  `tabName = "<tabs-id>-<tabName>"`, e.g. `"tabs-today"`.
- Keep tabs to **4–5 max**; mobile bottom bars get cramped beyond that.
- Module UI functions are **camelCase, descriptive** (`playerExplorerUI`), per
  `docs/CONVENTIONS.md §2`. Static reference tables are passed straight into the UI
  function as plain data frames (e.g. to build a `f7VirtualList` of players at UI time);
  reactive state is wired in the server (`docs/ARCHITECTURE.md §4`).

---

## 2. Layout building blocks

| Component        | Use for                                                  |
|------------------|----------------------------------------------------------|
| `f7Tab`          | a screen reachable from the bottom tab bar               |
| `f7Card`         | the default container for a chunk of content             |
| `f7ExpandableCard` | a card that opens fullscreen (great for player detail) |
| `f7List`         | rows of items (player lists, stat rows)                  |
| `f7Grid` / cols  | side-by-side layout (use sparingly on phones)            |
| `f7Block`        | generic padded content block                             |
| `f7Sheet`        | bottom sheet for filters / scoring settings              |
| `f7Searchbar`    | the search input pattern for player lookup               |

Mobile layout rule: **stack vertically, scroll, and lean on cards.** Avoid wide tables
and multi-column desktop layouts.

---

## 3. Inputs (Framework7-native > generic Shiny)

Prefer these so the app feels native:

- `f7Text`, `f7Select`, `f7Picker`, `f7SmartSelect` — text / dropdowns
- `f7Toggle` — booleans (e.g. "regular season only")
- `f7Slider` / `f7Stepper` — numeric ranges (e.g. trailing-games window)
- `f7Radio`, `f7CheckboxGroup` — choices (e.g. scoring profile)
- `f7Searchbar` — player search
- `f7Button` / `f7Segment` — actions and segmented toggles

Read selected values server-side via `input$<id>` as usual. Update them with
`updateF7*` functions.

---

## 4. Server-side UI feedback

shinyMobile gives native overlays — use them for feedback instead of inline text:

```r
f7Toast(text = "Data updated", session = session)
f7Dialog(id = "confirm", title = "Reset roster?", type = "confirm")
f7Notification(text = "...", title = "...")
f7Sheet(id = "filters", ...)          # toggle with updateF7Sheet()
```

Use a **toast** for transient confirmations, a **dialog** for confirmations/choices, a
**sheet** for filter/settings panels.

---

## 5. Tables

- **Simple, short tables** → `f7Table` (native look).
- **Sortable / longer / interactive tables** (rankings, leaderboards) →
  **`reactable`** via `reactableOutput()` / `renderReactable()`. It paginates and sorts
  client-side and scrolls fine on mobile.
- **Color-scaled columns** (the fantasy-value heatmap) → **`reactablefmtr::color_tiles()`**
  in a `colDef(cell = ...)`, exactly as SlamStats colors its standings:

  ```r
  avg_for = colDef(
    name = "Pts F", maxWidth = 60,
    cell = reactablefmtr::color_tiles(
      data, number_fmt = scales::label_number(accuracy = 0.1),
      colors = RColorBrewer::brewer.pal(7, "RdYlGn"),
      opacity = 0.8, box_shadow = TRUE
    )
  )
  ```
- **Cells can render images** (team logos, headshots): a `colDef(cell = function(value)
  { if (grepl("^http", value)) tags$img(src = value, ...) else value })`.
- **Dark-mode re-theme (the `render_table` trick).** reactable doesn't follow the F7
  dark toggle by itself. SlamStats sets a global theme from F7 CSS variables and forces
  rendered tables to redraw. Mirror this in `server.R`:

  ```r
  observe({
    options(reactable.theme = reactable::reactableTheme(
      backgroundColor = ifelse(input$dark == "dark", "var(--f7-list-strong-bg-color)", "#fff"),
      stripedColor    = ifelse(input$dark == "dark", "var(--f7-text-editor-bg-color)", "rgb(0,0,0,0.03)"),
      borderColor     = ifelse(input$dark == "dark", "var(--f7-list-item-border-color)", "rgb(230,230,230)")
    ))
    r$render_table <- r$render_table + 1          # bump the shared signal
  }) |> bindEvent(input$dark)
  ```
  Then every `renderReactable` starts with `req(r$render_table)` so it re-runs on toggle.
- Debounce the search box before filtering a big table (or use the client-side
  `f7VirtualList` searchbar below, which needs no server round-trip at all).

---

## 6. Long lists, search, clicks & lazy rendering (SlamStats patterns)

These four patterns come straight from SlamStats and are how the player/team screens
stay fast and feel native. They lean on a little custom JS in `www/` — keep that JS
small and well-commented.

### 6a. `f7VirtualList` + client-side searchbar (no server round-trip)

A long player list is an `f7VirtualList` of `f7VirtualListItem`s built **at UI time** from
the static player-index table. The `f7Searchbar` filters it **in the browser** via a CSS
selector — the server never sees a keystroke:

```r
f7Searchbar(
  id = ns("player_search"),
  options = list(searchContainer = paste0("#", ns("player_list")),
                 searchIn = ".item-title")     # match against item titles
),
f7VirtualList(
  id = ns("player_list"), strong = TRUE, dividers = TRUE, mode = "media",
  items = lapply(seq_len(nrow(player_index)), function(i) {
    f7VirtualListItem(
      id = player_index$athlete_id[i],         # carry the id for click handling
      title = player_index$athlete_display_name[i],
      media = img(src = player_index$athlete_headshot_href[i]),
      href = "#", routable = TRUE
    )
  })
)
```

### 6b. Bridging native clicks to Shiny (`Shiny.setInputValue`)

`f7VirtualList` items and `f7ExpandableCard`s don't report clicks to Shiny on their own.
SlamStats captures them in `www/clickEvents.js` and pushes an input value:

```js
// www/clickEvents.js
$(document).ready(function () {
  // expandable card → input$expandable_card_clicked
  $(".card-expandable:not(.card-opened)").click(function () {
    Shiny.setInputValue("expandable_card_clicked", $(this).attr("id"));
  });
  // virtual-list item → input$<listId>_clicked  (priority:"event" so repeat clicks fire)
  $(".virtual-list").on("click", ".item-link", function () {
    var parentId = $(this).closest(".virtual-list").attr("id");
    Shiny.setInputValue(parentId + "_clicked", $(this).attr("id"), {priority: "event"});
  });
});
```

The server then reads `input$player_list_clicked` / `input$expandable_card_clicked` like
any input. (The reverse direction — R → JS — uses
`session$sendCustomMessage()` + `Shiny.addCustomMessageHandler()` in
`www/customMessage.js`.) Include these scripts once via `tags$script(src = ...)` in the
`f7TabLayout` (see §1).

### 6c. Build detail views server-side with `f7Popup`

On a player click, SlamStats builds an `f7Popup(page = TRUE, ...)` in an `observe()` and
fills it with the card, table, and chart outputs — created on demand rather than living
in the DOM up front:

```r
observe({
  f7Popup(
    id = paste0("player_popup", input$player_list_clicked),
    title = this_player$athlete_display_name[1], page = TRUE,
    f7BlockTitle("Player Overview"),
    reactableOutput(ns("player_overview")),
    echarts4rOutput(ns("player_stats_chart"))
  )
}) |> bindEvent(input$player_list_clicked)
```

### 6d. Lazy rendering + progress

- **Render only when visible.** Team charts live inside `f7ExpandableCard`s and only
  render when the card opens, gated on the shared click signal:
  `req(r$expandable_card_clicked == paste0(id, "-card"))`. This avoids rendering dozens of
  charts up front.
- **Show progress for slow work** with `f7Progress` nudged via `updateF7Progress(id =
  session$ns("progress"), value = ...)` as a computation proceeds.
- **Deep-link to a tab** from a URL with
  `parseQueryString(session$clientData$url_search)` + `updateF7Tabs(id = "tabs", selected
  = ...)`.

---

## 7. Charts

Use **`echarts4r`** for all charts and **`r2d3`** for any bespoke D3 visual — this is the
SlamStats house style and what our `renv.lock` pins (`echarts4r` 0.4.5, `r2d3` 0.2.6).
We do **not** use `apexcharter`.

Set the font once in `global.R` so every chart inherits it:

```r
echarts4r::e_common(font_family = "Barlow Condensed")
```

UI/server wiring uses `echarts4rOutput()` / `renderEcharts4r()`:

```r
# UI
echarts4rOutput(ns("trend"))
# server — player fantasy-points trend line
output$trend <- renderEcharts4r({
  df |>
    e_charts(game_date) |>
    e_line(fantasy_pts, name = "Fantasy pts", color = "#E2552C") |>
    e_tooltip(trigger = "axis") |>
    e_legend(show = FALSE)
})
```

The echarts4r idioms SlamStats relies on (reuse these recipes):

| Chart            | Recipe                                                                 |
|------------------|------------------------------------------------------------------------|
| **Trend line**   | `e_charts(game_date) |> e_line(fantasy_pts)`                            |
| **Bar (ranked)** | `e_charts(name) |> e_bar(value) |> e_flip_coords()` for horizontal bars |
| **Radar** (compare) | `e_charts(stat) |> e_radar(value, max = 1, areaStyle = list(...))` — exactly how SlamStats draws a player's percentile profile across PTS/REB/AST/STL/BLK/TO |
| **Scatter**      | `e_charts(x) |> e_scatter(serie = y, bind = label)` with a JS `e_tooltip(formatter = htmlwidgets::JS(...))` |

- Pass per-category percentiles (0–1) into the radar with `max = 1`; split long axis
  labels onto two lines (SlamStats does `"Turn-\novers"`) so they don't clip on phones.
- For a single headline number (e.g. percentile rank) `f7Gauge` is fine.
- **Custom D3** (e.g. a standings bump chart) → `r2d3::renderD3({ r2d3(data, script =
  "www/bumpchart.js", css = "www/bumpchart.css") })`, with `d3Output(ns(...))` in the UI.

Keep charts simple and tall-friendly; avoid tiny multi-series legends on phone width.

---

## 8. Theming

- One accent color set in the `app_config` list → `f7Page(options = app_config)`. Accept
  a brand hex (`#E2552C`) or a Framework7 named color from `getF7Colors()` (SlamStats uses
  `"deeporange"`). The runtime color picker only offers `getF7Colors()` names.
- **Runtime theme switching (the Preferences panel).** SlamStats exposes `theme`, `dark`,
  and `color` as `f7Radio`s in a right-side `f7Panel`, and applies them live with
  `updateF7App()`:

  ```r
  observe({
    updateF7App(options = list(theme = input$theme))
  }) |> bindEvent(input$theme, ignoreInit = TRUE)

  observe({
    updateF7App(options = list(dark = identical(input$dark, "dark")))
  }) |> bindEvent(input$dark, ignoreInit = TRUE)
  ```
  Use `ignoreInit = TRUE` so the app doesn't re-theme on first load. Remember reactable
  needs the separate `render_table` redraw (see Tables) — `updateF7App` alone won't
  restyle already-rendered tables.
- Team colors are available in the cached team directory (`team_color`,
  `team_alternate_color` from hoopR) — use them for player/team cards. Inline styles may
  reference **F7 CSS variables** (`var(--f7-...)`) so they track theme/dark automatically.
- Put all custom CSS in `www/custom.css` and include it once via
  `tags$link(rel = "stylesheet", href = "custom.css")`; **no inline styles scattered in
  modules** (small variable-based inline styles for one-off layout are tolerated, as in
  SlamStats, but anything reusable belongs in the stylesheet).
- Support dark mode (`dark = TRUE`); test both.

---

## 9. PWA (installable app)

```r
# 1. allowPWA = TRUE in f7Page  (already set)
# 2. generate assets once:
charpente::set_pwa("/path/to/app")     # creates manifest + service worker in www/
```

This lets users add Hoop Vision to their phone home screen and launch it fullscreen.
Provide proper icons in `www/`. Revisit before the deploy phase.

---

## 10. Mobile do / don't

**Do**
- Design for ~390px width first.
- Use cards + vertical scroll.
- Use native overlays (toast/sheet/dialog) for feedback.
- Test on an actual phone over local network early.

**Don't**
- Don't build dense desktop tables that need horizontal scrolling.
- Don't use more than ~5 bottom tabs.
- Don't block the UI thread on data work — the cache is already built offline.
- Don't hand-roll HTML when an `f7*` component exists.

---

## 11. Reference

- shinyMobile docs: https://rinterface.github.io/shinyMobile/
- Framework7: https://framework7.io/docs/
- Component gallery: see the shinyMobile site's gallery + `?` help for each `f7*`.
- echarts4r docs: https://echarts4r.john-coene.com/
- reactable / reactablefmtr: https://glin.github.io/reactable/ ·
  https://kcuilla.github.io/reactablefmtr/
- r2d3 docs: https://rstudio.github.io/r2d3/
- **The living reference is the SlamStats source** in the workspace
  (`/Users/ezcz/Dropbox/gavin/SlamStats`): `R/player.R` (virtual list + popup + radar),
  `R/ranking.R` (color_tiles + r2d3 bump chart), `R/team.R`/`R/team_stats.R` (nested
  modules + lazy charts), `app.R` (skeleton, Preferences panel, shared `r`), and
  `www/clickEvents.js` / `www/customMessage.js` (the JS bridges).
