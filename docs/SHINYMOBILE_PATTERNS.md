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

```r
# R/ui.R
ui <- f7Page(
  title = "Hoop Vision",
  allowPWA = TRUE,                       # enables installable PWA
  options = list(
    theme = "auto",                      # "ios" | "md" | "auto"
    dark = TRUE,
    color = "#E2552C"                    # AthlyticZ / Hoop Vision accent
  ),
  f7TabLayout(
    navbar = f7Navbar(title = "Hoop Vision"),
    f7Tabs(
      id = "tabs",
      mod_today_ui("today"),
      mod_player_explorer_ui("players"),
      mod_rankings_ui("rankings"),
      mod_compare_ui("compare"),
      mod_team_ui("team")
    )
  )
)
```

- **`f7Page`** is the root. `allowPWA = TRUE` wires up the PWA path.
- **`f7TabLayout`** + **`f7Tabs`** gives the bottom tab bar — our primary navigation.
- Keep tabs to **4–5 max**; mobile bottom bars get cramped beyond that.

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
  client-side and scrolls fine on mobile. Style columns (color scales for fantasy
  value, team logos via `athlete_headshot_href`).
- Debounce the search box before filtering a big table.

---

## 6. Charts

Use **`apexcharter`** — it's in shinyMobile's suggested stack and renders crisply on
mobile.

```r
# UI
apexchartOutput(ns("trend"))
# server
output$trend <- renderApexchart({
  apex(data = df, type = "line", mapping = aes(game_date, fantasy_pts))
})
```

Good chart choices for this app:
- Player **trend** line (fantasy pts per game over time).
- **Bar** comparison across players.
- **Radar** for category profiles (PTS/REB/AST/STL/BLK) in compare view.
- `f7Gauge` for a single headline number (e.g. percentile rank).

Keep charts simple and tall-friendly; avoid tiny multi-series legends on phone width.

---

## 7. Theming

- One accent color set in `f7Page(options = list(color = ...))`.
- Team colors are available in the cached team directory (`team_color`,
  `team_alternate_color` from hoopR) — use them for player/team cards.
- Put all custom CSS in `www/custom.css` and include it once; **no inline styles in
  modules**.
- Support dark mode (`dark = TRUE`); test both.

---

## 8. PWA (installable app)

```r
# 1. allowPWA = TRUE in f7Page  (already set)
# 2. generate assets once:
charpente::set_pwa("/path/to/app")     # creates manifest + service worker in www/
```

This lets users add Hoop Vision to their phone home screen and launch it fullscreen.
Provide proper icons in `www/`. Revisit before the deploy phase.

---

## 9. Mobile do / don't

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

## 10. Reference

- shinyMobile docs: https://rinterface.github.io/shinyMobile/
- Framework7: https://framework7.io/docs/
- Component gallery: see the shinyMobile site's gallery + `?` help for each `f7*`.
