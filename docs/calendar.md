# Run calendar: decisions

> **Updated 2026-11-17.** The two looks now style the whole app, not only the Calendar tab, and units, sharing and shoe photos were added. See "Changes on 2026-11-17" at the end; where that section disagrees with the text above it, it wins.

Built 2026-11-16 from `docs/calendar-claude-code-prompt.md`. Visual references: `docs/design/calendar-soft.html` and `docs/design/calendar-contrast.html`. Interaction pattern borrowed from ballet closet's on-screen calendar (`renderCalendar`, `openDay`, `getStreak`): month grid, tap a day to open it in a dialog, "+N" on days with several records, week streak.

## What it is

A fourth tab, **Shoes · Calendar · Runs · Retired**. It is a second view of `S.runs`; it adds no record type and no field to a run. The app is still one `index.html` with inline CSS and JS, no build step, no dependencies, data in `localStorage` under `shoetracker.v1`.

## Data (additive only)

| Field | Values | Default |
|---|---|---|
| `S.calTheme` | `'soft'` \| `'contrast'` | `'soft'` |
| `S.weekStart` | `0` (Sunday) \| `1` (Monday) | `0` |
| `shoe.colorIdx` | `0`–`5` | assigned |

- The migration lives in `load()`, so fresh state, old state and restored backups all go through it. A backup is still plain `JSON.stringify(S)`.
- A shoe without a valid `colorIdx` gets the lowest index not used by another active shoe, falling back to `position % 6`. New shoes get one the same way; the shoe dialog has six swatches ("Calendar color") to change it.
- A retired shoe does not reserve its color, so a retired and an active shoe can share one.
- Settings has "Calendar style" and "Week starts on".

## Pure functions

Between `/* pure:start` and `/* pure:end */` in `index.html`. Data in, data out: no DOM, no reads of `S`.

- `monthGrid(ym, weekStart)`: weeks of 7 entries, a `YYYY-MM-DD` string or `null` outside the month.
- `dayIndex(runs)`: Map of date → `{ km, runs, mainShoeId }`. Main shoe is the one with the most km that day; ties go to the most recently logged run (`t`).
- `monthSummary(runs, ym)`: `{ km, count, longest, byShoe }`. `count` is runs, not days.
- `weekTotals(runs, ym, weekStart)`: one `{ from, to, km }` per grid row for the **full 7-day week**, including days in the neighbouring month. These are training weeks, so they do not add up to the month total.
- `weekStreak(runs, todayStr, weekStart)`: consecutive weeks with a run, counting back from this week, or from last week if this week has none yet.
- `freeColor`, `assignColors`: the color rule above.

Dates are built from year, month and day numbers at local midnight (`pd`, `iso`, `addDays`). `YYYY-MM-DD` is never passed to `new Date(str)` alone, which would shift by the UTC offset.

`tests.html` fetches `index.html`, evaluates that block and runs plain assertions (48) against the check data and edge cases. Open it over http, for example `python -m http.server` then `/tests.html`.

## The tab

Top to bottom: month header (previous / next; tapping the month name returns to the current month), month stats (km, runs, week streak when it is 2 or more), weekday row and grid, shoe legend, weekly km bars. `calMonth` is not persisted.

- Run day: filled with the main shoe's color, date and day total. Two or more runs: "+1", "+2" … in the top right.
- No run, today or earlier: bordered empty cell. Future: dimmed, still tappable. Today is marked. Cells outside the month are empty spans, not focusable.
- Tapping a day opens the day dialog: full date, day total, one row per run (color, shoe, type, km). A row opens the existing run dialog; "+ Log run on this date" opens it with the date prefilled (`openRun(id, date)`; without a date it behaves as before).
- An empty month shows the full grid with zero stats. With no shoes at all, the existing "Add your first pair" state is shown.
- A run whose shoe was deleted uses palette index 0 and the name "(deleted shoe)".

## Two looks

`cal` plus `cal-soft` or `cal-contrast` go on `<body>` only while the Calendar tab is active. Each look overrides the app's CSS variables, so the header, tabs and floating buttons follow it. Both palettes are fixed and ignore `prefers-color-scheme`; the other tabs keep the light / dark behavior. `<meta name="theme-color">` follows the look (`#FFFFFF`, `#0A0A0A`, original elsewhere).

- **Dialogs keep the app's styling.** The app variables are declared on `:root, dialog`, so a dialog resets them even when `<body>` carries a look.
- The style toggle sits in the header next to settings and is only shown on the Calendar tab. It flips `S.calTheme`, saves and re-renders.
- **Soft**: white, rounded, pale fills, serif italic month name (system Georgia; no font file).
- **Contrast**: black, square corners, Archivo Black and Space Mono, solid color blocks, volt accent.
- The shoe palette (six fills per look) is `CAL_PAL` in `index.html`.

## Fonts

`fonts/archivo-black-400.woff2` and `fonts/space-mono-700.woff2`, Latin subsets from Google Fonts, both OFL (`fonts/OFL.txt`). Declared with `@font-face` and `font-display: swap`; the browser only fetches them when the Contrast look is shown. Both are in `SHELL` in `sw.js` and the cache name is `shoes-v2`, so they work offline. Nothing is loaded from a CDN.

## Where this differs from the reference files

- **Shoe code.** The reference shows "PG", "NB", "VF", "SG". The app uses the existing `initials()` ("P4", "N5", "V3", "S6"), as the prompt specifies.
- **Km format.** The calendar always shows one decimal ("16.0", "112.0") to match the reference, through a calendar-only `km1`. `fmtKm` is unchanged, so the other tabs still show "16". A day total of 100 km or more is shown without the decimal so it fits the cell.
- **Month name size.** At 390px the reference itself overflows: "November" at 52px pushes the arrow buttons past the right edge. `fitMonth()` shrinks the month name until the row fits (47px for November at 390px, 40px at 360px, 46px for September at 390px). At 560px it is the full 52px. Contrast needs no shrink at 390px.
- **Space Mono weight.** The reference uses Space Mono 400 for legend names and week labels. Only the 700 file is shipped, so those are bold.
- **Streak.** The reference shows a 7-week streak; the check data alone gives 3.
- **Day cells at 360px** are 41.7px wide (seven columns with 6px gaps), under the 44px touch target, and 58px tall.

## Accessibility and motion

- Every day is a real `<button>` with a label such as "Sunday, November 15: 21.1 km, Vaporfly 3"; today has `aria-current="date"`.
- The grid is one tab stop (roving `tabindex`). Arrow keys move within the month; Enter or Space opens the day.
- `:focus-visible` rings in both looks. The style toggle and the Soft tabs have their hit area extended to 44px.
- In Contrast, the shoe code keeps shoes from being told apart by color alone.
- Month changes fade the grid in over 120ms, and not at all under `prefers-reduced-motion`.

## Not in this change

Share images of the calendar, planned runs, pace or duration, GPS or Strava import, run notes or photos, year view, goals.

## Changes on 2026-11-17

### One look for the whole app
- Soft and Contrast now style every tab and every dialog. The old light / dark styling and `prefers-color-scheme` are gone.
- `<body>` always carries `look-soft` or `look-contrast` (these replace `cal-soft` / `cal-contrast`), plus `cal` while the Calendar tab is open. Soft tokens sit on `:root`; `body.look-contrast` overrides them. The `:root, dialog` reset is removed, so dialogs follow the look.
- The setting is still stored as `S.calTheme` (kept for old backups). The header toggle and the "Style" setting are shown on every tab.
- Shoe cards, the runs list and dialogs use each shoe's calendar color (thumbnail fill, dot).
- Dialogs are full-width bottom sheets on phones.

### Distance, not months
- The months-based age warning and its setting are removed. `S.ageMonths` is left in the data and ignored. Cards still say how old a shoe is.
- `S.unit` is `'km'` or `'mi'`. **Everything is stored in km**; miles exist only on screen (`kmTo`, `kmFrom` in the pure block; `fmtD`, `d1` for display).
- Distance inputs use `setD` / `getD`: a field that was not edited hands back the exact stored km, so saving in miles never drifts the data.
- `S.limitKm` (default 700) is "Replace shoes at" in Settings: the limit new shoes start with. Changing it offers to apply the new limit to the current shoes too.
- The CSV export stays in km.

### Share
- A share icon in the header, on the Calendar tab only. `shareCanvas()` draws the month shown, in the current look, onto a 1080×1920 canvas with plain canvas calls (no library), mirroring the on-screen layout.
- The dialog previews the picture and offers "Save image" and, where the browser supports sharing files (`navigator.canShare`), "Share". Nothing is uploaded by the app.

### Calendar days: color block or shoe photo
- `S.calCell` is `'block'` (color and distance, the default) or `'photo'`. In photo mode a run day shows the main shoe's photo instead of the distance: on the shoe's pale fill in Soft, inside a border of the shoe's color in Contrast. A shoe without a photo falls back to the color block.
- Photos are applied through one CSS rule per shoe (`<style id="pics">`), so a photo is not repeated in every cell.
- **Cut-outs.** `cutout()` is ported from ballet closet's `cutout.js`: it removes a plain, even background by flood-filling inwards from the border, entirely on the device, with no model and no download. It returns nothing for busy backgrounds (a floor, a room), and the original photo is kept. A picture that is already transparent is only cropped.
- Choosing a photo tries the cut-out automatically; "Cut out" / "Original" switches between the two before saving. The saved photo is one image: `shoe.photo` (a PNG of at most 240px when cut out, else the square JPEG) with `shoe.photoCut` true or false. Both fields are additive.
