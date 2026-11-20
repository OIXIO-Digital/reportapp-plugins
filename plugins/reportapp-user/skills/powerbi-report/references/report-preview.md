# The Power BI display page — `window.reportPreview`

What an assistant can read and do on a ReportApp page that embeds a Power BI report (`/reports/display?elementId=<id>`, and the hybrid display page). `window.reportPreview` is the page's own API for this: it reads the embedded report through the Power BI JavaScript client the page already runs, as the signed-in user, and changes the view the way the user's own clicks would.

It covers **what the user sees** — report, page, visuals, filters, slicer selections, what they clicked, a visual's numbers as shown. It does **not** query the semantic model: questions the screen can't answer go to the assistant's own Power BI tooling (see the skill's `references/fabric-iq.md`). The report itself is never edited.

---

## 1. Availability

- **The page:** `/reports/display?elementId=<id>` or the hybrid display page. Not `/reports/displaywebsite` (a published-to-web iframe — nothing to read).
- **The build:** probe `typeof window.reportPreview?.setSlicer === 'function'`. If `reportPreview` is missing the deployed build predates it; say so and work from a screenshot instead. Never fall back to the page's Power BI token or the Power BI REST API.
- **Any signed-in user.** Nothing is admin-gated; it reads only what the user's own report shows.
- **Loaded:** every call waits for the report to load (30 s). `await reportPreview.ready()` is there for an explicit wait.

All calls are `async` (return promises) except `selection()`.

**Naming a visual** (`visualData`, `slicerValues`, `setSlicer`, `clearSlicer`): its `name`, its `label`, or — when no visual has that label — its title (slicers usually hide theirs: `'Time dimension'`). A shape or textbox carrying the same heading as the data visual inside it gives way to that visual (`'Klientide arv'` is the chart, not its background shape). A label or title still shared by several visuals is refused with the candidates (`Several visuals match "Müügitulu": <visual-name> (lineStackedColumnComboChart), …. Use the name.`) — pick one and pass its `name`.

---

## 2. Output safety

Tool output is truncated at roughly 4 KB. `filters()` with raw filters, `filters({ visuals: true })` and `visualData()` on a big table exceed that. Map to what you need in the same call:

```js
const f = await reportPreview.filters();
JSON.stringify({
  report: f.report.map(x => x.text),
  page: f.pageFilters.map(x => x.text),
  slicers: f.slicers.map(s => s.title + ': ' + s.filters.map(x => x.text).join('; '))
});
```

Never return the page's HTML, hrefs other than `link()`'s, or the report container's attributes.

---

## 3. Reading

### `info()`

```js
await reportPreview.info()
// { elementId, name, sharedReport, workspaceId, reportId, datasetId, datasetIdFrom,
//   powerBiUrl: 'https://app.powerbi.com/groups/<ws>/reports/<rep>/<activePageName>',
//   reportAppUrl, activePage: { name, displayName }, pages: [{ name, displayName, isActive }] }
```

- `pages` lists the pages the report shows; pages its author hid are left out (unless one is active).
- `powerBiUrl` is the report's address in the Power BI service — the form Fabric IQ's `ResolveFabricItem` accepts. `workspaceId` falls back to `me` for a report shared from someone's My workspace.
- `datasetId` is the report's semantic model, so you can skip discovering it:
  - `datasetIdFrom: 'report'` — the server passed the report's own model;
  - `datasetIdFrom: 'refresh setup'` — taken from the element's dataset-refresh setting, which an admin enters. It is almost always the report's model, but confirm it if a model query disagrees with the screen;
  - `null` — neither is available: resolve the model from `powerBiUrl`.

### `summary({ visuals, rows, pages })`

**One read-only call for analysis.** It covers what the user sees on the active page and changes nothing: no page, filter or slicer change, no toast.

```js
await reportPreview.summary()
// { warnings: [{ kind: 'dataEndsEarly', visuals: ['Müügitulu', 'Ärikasum (-kahjum)', …],
//                text: 'The data ends with IV-24 (2024-12-31) but the period filter runs to 2026-10-15: …' }, …],
//   report: { elementId, name, workspaceId, reportId, datasetId, datasetIdFrom, powerBiUrl },
//   page: 'Võtmenäitajad', pages: [...],
//   refreshed: { at: '2026-10-14T22:05:31Z', source: 'semantic model refresh history' },
//   filters: ['Financial Statement[FinRepTypeName] in ("P&L", "KPI")', …],
//   slicers: [{ slicer: 'Time dimension', field: 'Date set[Date set Fields]', kind: 'grouping', selection: "Date set[Date set Fields] in (\"'Date'[Quarter]\")" },
//             { slicer: 'Company filter', kind: 'filter', selection: 'Company[Company] in ("<company>")' }, …],
//   visuals: [
//     { name, label: 'Müügitulu', type: 'cardVisual', categories: [], rows: 1,
//       measures: [{ name: 'Actual', unit: 'number', value: 1234567 }, { name: 'Budget fulfillment%', unit: 'percent', value: 99 }, …] },
//     { name, label: 'Müügitulu', type: 'lineStackedColumnComboChart', categories: ['Kvartal'], rows: 4,
//       measures: [{ name: 'Actual', unit: 'number', points: 4, first: 'I-24: 1100000', last: 'IV-24: 1050000',
//                    min: 'III-24: 900000', max: 'I-24: 1100000' }, …] },
//     { name, label: null, type: 'card', categories: ['PBI Refresh'], rows: 1, measures: [], text: 'PBI Refresh: 14. Oct 2026 01:05' } ],
//   clicked: null | { visual, points: [identity…] },
//   otherPages: [{ page: 'Turnover', visuals: ['Turnover and average number of employees', 'Turnover', …] }, …] }  // see below
```

- **`warnings`** come first, and they are computed in code, so you only have to **repeat** them, never work them out. Each is `{ kind, visuals?, text }`:

  | `kind` | When |
  |---|---|
  | `selectionActive` | The user has a point selected: the other visuals are filtered or **highlighted** to it, and a highlighted visual can still show unfiltered totals. |
  | `refreshFailed` | The newest refresh attempt failed; the numbers are from the last successful one. |
  | `staleData` | The last completed refresh is more than 7 days old. |
  | `noRefreshInfo` | No refresh time is known (`refreshed.note` says why). |
  | `truncated` | A visual has more rows than were read; its figures cover only those rows. |
  | `notExported` | Power BI returned no rows for a visual that may show data. |
  | `exportFailed` | A visual could not be read. |
  | `nonFinite` | A measure has `∞`/`NaN` values (division by zero), left out of its min/max. |
  | `dataEndsEarly` | A visual's last period (from its time axis) ends more than 31 days before the period filter's end, or before today if that is earlier. There is no data loaded after it: not a decline. |

  Visuals with the same last period are grouped into one warning.
  - **Time axes:** read from category labels: quarters (`IV-24`, `Q4 2024`), months (`2-24`, `2024-02`, `Feb 2024`, `veebr 2024`), years, and dates (`14.01.20`, `2024-01-14`).
  - **Ambiguous labels:** `2-24` (could be a band) and `2024` (could be a number) count only in a column named like time (Kuu, Month, Kvartal, Aasta, Kuupäev, Date, Period…).
  - **Weeks** (`3/20`) are not read.
  - **Relative date filters** ("in the last 12 months") count as ending today: they are relative to the calendar, not to the data.
  - **No date filter** (none at all, or a numeric "relative day" column): `dataEndsEarly` can't be computed.
- **Not detectable:** row-level security. The embedded report simply returns fewer rows, with no signal in the API, so no warning will ever flag it — the skill's RLS rule covers it.
- **`visualFilters`** on a visual lists its own filters (e.g. a card limited to `Financial Statement[Level1] in ("Müügitulu")`): its numbers are for that subset only.
- **Visuals:** the data visuals on the active page, top to bottom; slicers and decoration are left out. `{ visuals: ['Müügitulu', '<name>'] }` limits them by name or label (by title only for a visual without a label, since hidden titles are often leftovers from copies). `{ rows }` caps each export (default 500).
- **Measures:**
  - a column counts as a measure when every value in it is numeric, or when it is entirely blank (a card showing `--` → `value: null`);
  - the other columns are the `categories` (the axis);
  - `first` / `last` / `min` / `max` give the category each point is at;
  - a one-row visual (a card) gives `value` instead;
  - a visual whose export fails has `error` in place of measures;
  - **an export with no rows** gives `rows: 0`, the `columns`, no measures, and a `note`. Some visuals show data that their export doesn't carry: e.g. the hierarchical COMPARISON finance matrix (`Level1…Level6`) exports only its header. Read those through a model query, or describe them from a screenshot. Never report them as empty;
  - **`nonFinite`** counts values like `∞` (Change % on a zero opening balance) or `NaN`. They keep the column a measure but are left out of `first`/`last`/`min`/`max`. Mention them when they matter.
- **`scale`** comes from the measure's format string: `thousands` / `millions` / `billions` when it divides the shown value (`#,0,`, `#,0.0,,`). The export carries the raw value.
- **`unitHint`** (on a measure or a visual) comes from the measure name or the visual's title: `EUR`, `EUR, thousands`, `percent` ("Väärtus laos (€)", "tuh €", "Budget fulfillment%").
- **`blanks`** counts rows where the measure had no value: "no rows matched", not zero. Blanks are left out of first/last/min/max.
- **`unit`** comes from the measure's format string, or a `%` on its values: `percent`, a currency (`EUR`, `USD`, `GBP`), `number`, or `null` when the export carries no format. The export only includes format strings for measures with a dynamic format; for the others, the model's format string (a model query) is the authority.
- **Only the open page.** Power BI exports only the visuals of the page on screen: every export from another page fails (`ExportDataFailed`). To read another page's numbers, open it (`goToPage`, a view change: ask) or query the model.
- **`otherPages`** lists the other visible pages and what their data visuals are about. It is built from the layout alone, with no navigation and no export (about 60 ms for 6 pages).
  - It is added automatically when the open page has no data, e.g. a MENU landing page (images and navigation buttons only), and on request with `{ pages: true }`.
  - `unnamed: n` means n visuals there have no readable name yet (dynamic titles; see `visuals()`), so the list is not complete.
  - Use it to tell the user where the answer is ("the Turnover page has *Turnover* and *New / Active / Resigned* tables"), then offer to open that page.
- **`hiddenTitle`** is added for a visual without a label: its configured but hidden title. It is often an accurate auto-generated title ("New Employees and Resigned Employees by Month"), but can be a leftover from a copied visual. Check it against the visual's measures before using it as a name.
- **`refreshed`** is the semantic model's last *completed* refresh, from the model's refresh history, read through the page's own dataset-refresh endpoint. Example: 2024-03-27, with `latestAttempt: { status: 'Failed', … 2024-08-19 }`. The data is years old and the last refresh failed: say so before any analysis.
  - It needs a `datasetId` and access to that history.
  - `latestAttempt` is added when the newest run didn't complete (failed, or still running).
  - Otherwise it falls back to a visual on the page that shows a refresh time (`source: 'visual "…"'`).
  - Failing both, it returns `{ at: null, note }` saying why: the model has no refresh history (DirectQuery, or only ever published), the history isn't readable with the user's access, or the model isn't known. A page in `otherPages` may show the time (e.g. an *Andmed uuendatud* card on a *Laoseis (täna)* page).
  - It is the model refresh, not the source system's load time.
- **`slicers[].kind`:**
  - `filter`: a row filter.
  - `grouping`: a field-parameter slicer choosing the breakdown, e.g. Kuu / Kvartal / Aasta, or a dimension such as `'Ledger Account'[Account Long Name]`.
  - `measures`: a field-parameter slicer choosing which measures the visuals show, e.g. `'#Measures'[Balance]` ("Selected measures").

  Neither `grouping` nor `measures` limits which rows are included.
- **`slicers[].field`** is the column the slicer filters (`Data origin[AppName]`). Use it to tell slicers apart when their titles don't: one report has eight slicers titled `Business Line filter`, each on a different field (AppName, Dimension 1–7). Their on-screen headers are display names that the API doesn't expose.
- **Cost:** one export per visual. On a page with 8 data visuals it took about 2.5 s and returned about 4.7 KB. Limit it with `visuals`, or map it down in the same script.

### `visuals(page?)`

The visuals on a page (default: the active one), top to bottom then left to right, hidden ones left out:

```js
await reportPreview.visuals()
// [{ name: '<visual-name>', type: 'lineChart', title: 'Müügitulu', titleShown: false, label: 'Klientide arv', x, y, width, height }, …]
```

- `name` is the stable id — use it once you know which visual.
- **`label` is the heading the user reads** for the visual — use it to find "the Müügitulu chart" and to cite. It is:
  1. the visual's own title when shown;
  2. else the shown title of the card, shape or textbox directly above it (a KPI card above a chart, bottom edge ≤ 40 px above, overlapping half its width), or of the shape/textbox it sits inside — whichever starts closer. A chart's title never labels the chart below it (on one Turnover page that put the line chart's heading on the ribbon chart);
  3. else `null` (unlabelled — describe it by type and position, or ask).
  Slicers never label other visuals.
- **`title` is the configured title text, shown or not** (`titleShown`). Report authors copy visuals and hide the title, so a hidden `title` is often wrong: on Võtmenäitajad four visuals have the title `Müügitulu`, among them the charts under *Klientide arv* and *Tootlikkus*. Never cite a hidden title. Placeholder titles (`Card`, `Line chart`, `Shape`, …) and whitespace-only titles count as no title.
- **Dynamic (fx) titles** are computed only when their page renders. Until then a visual reports its type's name ("Matrix", "Line and stacked column chart") as its title.
  - Labels on the open page wait for its render (up to 8 s), so they match the screen.
  - On pages that aren't open, such visuals have no label: `otherPages` counts them as `unnamed`. On one Finantsaruanne page, both "Müügitulu" visuals are like this.
- Labels repeat too (the Müügitulu card and the Müügitulu chart under it): pick by `type`, or by `name`.
- Slicer types: `slicer` (classic), `advancedSlicerVisual` (button/list slicer), `newTimeline…` (the custom Timeline; the type ends in a GUID).
- `shape`, `textbox`, `image`, `pageNavigator`, `actionButton` are decoration or navigation, not data.

### `filters({ page, all, visuals })`

The filters in effect on a page (default: the active one):

```js
await reportPreview.filters()
// { page: { name, displayName },
//   report:      [ filter… ],   // report level (filter pane, all pages)
//   pageFilters: [ filter… ],   // this page
//   slicers:     [{ visual, title, field: 'Company[Company]', type, filters: [ filter… ] }] }
```

Each filter is `{ text, dax, filter }`:

| Field | What |
|---|---|
| `text` | One readable line: `Company[Company] in ("<company>")`, `Date[Date] GreaterThanOrEqual 2024-01-01 and Date[Date] LessThanOrEqual 2026-10-14` |
| `dax` | An exact single-column DAX predicate for `CALCULATETABLE(…, KEEPFILTERS(<dax>))`: `'Company'[Company] IN {"<company>"}`, `('Date'[Date] >= DATE(2024, 1, 1) && 'Date'[Date] <= DATE(2026, 10, 14))`. **`null`** when no exact column predicate exists: filters on a measure, on a hierarchy, relative date/time ("in the last 3 months"), TopN, tuple filters. Express those yourself from `text` and `filter`. |
| `filter` | The raw Power BI filter object |

- Empty filter-pane cards and slicers with nothing selected are left out; `{ all: true }` includes them.
- `{ visuals: true }` adds `visuals: { <visualName>: { title, filters } }` — visual-level filters, e.g. a card limited to `Financial Statement[Level1] in ("Müügitulu")`. Large; map it down.
- **Dates:** raw `filter` values are UTC timestamps of the browser's local midnight (1 Jan 2024 in Tallinn is `2023-12-31T22:00:00.000Z`). `text` and `dax` already convert to the local calendar date — use them, not the raw value.
- **Field-parameter slicers** (e.g. *Time dimension*: Kuu / Kvartal / Aasta) filter a hidden `<name> Fields` (or bare `Fields`) column whose values are field references: `Date set[Date set Fields] in ("'Date'[Month]")`. That means "the visuals are grouped by month", not a row filter — don't put it in a DAX query as a predicate; group your query by that field instead. A measure field parameter (`#Measure collection Balance[Fields] in ("'#Measures'[Balance]", …)`) chooses which measures are shown: query those measures.

### `daxContext(visual?, { page, selection })`

Every filter in effect, as **exact** DAX for a model query, so a query can reproduce what the user sees without filters being copied by hand. Nothing is approximated: what has no exact form is listed, not guessed.

```js
await reportPreview.daxContext('<visual-name>', { selection: true })
// { page: 'Võtmenäitajad', visual: { name, title, type },
//   predicates:   [{ source: 'report' | 'page' | 'slicer "Company filter"' | 'visual' | 'selection in <visual>', text, dax }, …],
//   untranslated: [{ source, text }],      // no exact DAX form — the query won't match the screen until handled
//   fields:       [{ source: 'slicer "Time dimension"', kind: 'grouping' | 'measures', references: ["'Date'[Quarter]"] }],
//   complete:     true | false,            // false when anything is untranslated
//   calculateTableArgs: "KEEPFILTERS('Company'[Company] IN { \"<company>\" }),\n    KEEPFILTERS(…)" }
```

- **Sources:** report, page and slicer filters always; the visual's own filters when a visual is named; the clicked point with `{ selection: true }`.
- **Untranslated:** relative dates, measure filters, TopN, tuple and hierarchy filters, and a selection of several points over several columns (tuples). These stay in `untranslated`.
- **Selections:** one point gives one predicate per column. Several points on one column give `IN { … }`. A blank member gives `ISBLANK(col)`, because `= BLANK()` would also match 0 and `""`.
- **Field-parameter slicers** go to `fields`, not `predicates`: they choose the grouping or the measures, not rows.
- **Not visible to it:** slicer-to-visual interactions. Power BI doesn't expose an author switching a slicer off for one visual, so that slicer is still listed. Reproduce one visible number before relying on a query.

### `visualData(visual, { page, rows, underlying })`

The data behind a visual **as the user sees it** — every filter, slicer and cross-highlight applied — through Power BI's export:

```js
await reportPreview.visualData('<visual-name>', { rows: 50 })
// { visual: { name, title, label: 'Müügitulu', type }, page: 'Võtmenäitajad',
//   columns: ['Budget', 'Budget [format]', 'Actual', 'Actual [format]', …, 'Kuu'],
//   rows: [{ Budget: '120000', 'Budget [format]': '#,0', Actual: '115000', …, Kuu: '1-24' }, …],
//   truncated: false }
```

- Values are strings, unformatted (`1234567`, `99 %` for a percentage measure). A measure with a dynamic format string is followed by a `<measure> [format]` column holding it (`#,0`, `#,##0 %`).
- `rows` defaults to 1000 (`summary()`: 500). `truncated: true` means the visual has more rows than you got; summaries over them (min/max, first/last) cover only those rows.
- `{ underlying: true }` asks for the underlying rows; this needs Build permission on the semantic model and usually fails for report viewers (`ExportDataFailed`).
- Naming a shape, textbox or image is refused (`… is a textbox - it has no data`).
- Fails with a clear message when the report or tenant disallows data export. Slicers export too (a slicer exports its value list), but use `slicerValues` for that.

### `selection()`

What the user last clicked in a visual, or `null` when nothing is selected (clicking the same point again, or empty space, deselects). Not async.

```js
reportPreview.selection()
// { at, page: 'Võtmenäitajad',
//   visual: { name, title: 'Müügitulu', type: 'lineStackedColumnComboChart' },
//   dataPoints: [{ identity: { 'Date[Quarter]': '"II-24"' },
//                  values:   { '#Measures[Actual]': '250000' },
//                  dax: "'Date'[Quarter] = \"II-24\"" }] }
```

`dax` is an exact predicate for the clicked point's identity (`ISBLANK(...)` for a blank member). It is `null` when any part of the identity isn't on a plain column: a partial predicate would widen the point. `daxContext(…, { selection: true })` adds it to the other filters.

`visual.title` here is the configured title, which may be hidden and wrong; look up the visual's `label` in `visuals()` by `name` before naming it to the user.

### `slicerValues(visual, { page })`

The values a slicer offers, as its list shows them, from its export:

```js
await reportPreview.slicerValues('Business Line Code')   // ['', 'ADM', 'MRK', 'PL1', …]
```

- `''` is the blank member (shown as `--` or `(Blank)`).
- For a field-parameter slicer these are the **labels** (`Kuu`, `Kvartal`, `Aasta`), not the references `setSlicer` takes.
- For a Timeline the first column is every date (large) — don't call it there.
- `null` when export is disabled.

### `bookmarks()`

The report's bookmarks — views its author saved — flattened out of their groups:

```js
await reportPreview.bookmarks()
// [{ name: '<bookmark-name>', displayName: 'Restore Filters', group: null },
//  { name: '<bookmark-name>', displayName: 'Tegelik', group: 'Tehniline lk 6' }, …]
```

Some bookmarks are technical (button states, toggles) rather than views — judge by name and group.

### `link({ filters, slicers })`

A ReportApp URL that reopens this report with the view applied on load, through the page's `?filter=` (report-level filters) and `?slicer=` parameters:

```js
await reportPreview.link()
// { url: 'https://…/reports/display?elementId=<id>&slicer=…', filters: 0, slicers: 4,
//   omitted: ['page-level filters: …'] }
```

- **Default contents:** the report-level filters changed since the page loaded (the report's saved ones are not repeated), and every slicer with a selection — classic, button and Timeline — by title when unique, else by visual name.
- **`omitted`** lists what a link can't carry: page-level filters, and the active page when it isn't the report's default page (links open on the default page). Mention these when you hand the link over.
- **A Timeline is carried as a fixed range**, so a link made today still shows today's range next month.
- Pass `{ filters, slicers }` to link to another state — e.g. the one your answer describes: `slicers: [{ visualName: 'Business Line Code', filters: [<basic filter>] }]`.
- **Opening a link saves its state as the recipient's personal view** of the report (persistent filters). Say so when you share one.

---

## 4. Changing the view

Change the view only when the user asked for it or accepted your offer. Each change:

- shows a toast to the user — *"Assistent muutis vaadet: Business Line Code: … in ("MRK")"* (English: *"Assistant changed the view: …"*);
- is **saved as the user's personal view** of the report (persistent filters), like their own clicks;
- is covered by `undo()`: the first change of the session snapshots the view first.

### `goToPage(page)`

Opens a page by name or display name. Pages the report hides are refused. Unknown names fail with the list of pages.

### `setSlicer(visual, selection, { page })` / `clearSlicer(visual, { page })`

Sets a slicer on the active page (or `page`, which is opened first):

| `selection` | Example |
|---|---|
| list of values | `['MRK', 'PL1']` — classic and button slicers |
| range | `{ from: '2025-01-01', to: '2025-06-30' }` — Timeline / date slicers; `YYYY-MM-DD` is a local calendar date; either end may be left out |
| field references | `["'Date'[Quarter]"]` — field-parameter slicers |
| a Power BI filter (or array) | passed through |
| `null` | clears (same as `clearSlicer`) |

Returns `{ visual: { name, title }, page, filters }` with the slicer's state read back after the change — check it.

Refusals (nothing is changed):

- a value the slicer doesn't offer: `"Business Line Code" does not offer "XYZ". It offers: (blank), ADM, MRK, …`
- a label on a field-parameter slicer: `"Time dimension" is a field parameter: pass field references like "'Date'[Quarter]", not labels (Kuu, Kvartal, Aasta)…` — the current reference is in `filters()`; the others come from the parameter table (see the skill's `references/fabric-iq.md`), or ask the user to click the option.
- a visual that isn't a slicer.

**A value can exist and still empty the page:** the check is against the slicer's list, not against what the other filters leave. After a change, read a key visual with `visualData`; if it comes back empty, tell the user and offer `undo()`.

### `applyBookmark(name)`

Applies a bookmark by name or display name.

### `snapshot(label)` / `restore(id)` / `undo()`

```js
const s = await reportPreview.snapshot('before comparing');   // { id: 1, label, at }
await reportPreview.restore(s.id);
await reportPreview.undo();   // true, or false when there is nothing to undo
```

- `undo()` returns to the view **before the assistant's first change** (page, filters, slicers — the Timeline's relative period included). Then the next change starts a new undo point.
- Prefer `undo()` to the toolbar's **Taasta vaikesätted** (reset defaults): that one also throws away the user's own saved view. Never trigger the reset yourself.
- Snapshots and the undo point live in the page: **a reload loses them** (the changes themselves stay saved).

---

## 5. Not available

- Selecting or cross-highlighting data points from code (read the user's clicks with `selection()` instead).
- Changing a visual's fields or type, creating visuals — needs report edit rights.
- Drill down/up, focus mode.
- Report-, page- and visual-level filters other than slicers (`setFilter`) and sorting — planned, not built.
