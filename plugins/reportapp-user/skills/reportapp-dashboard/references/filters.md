# Filters — `tablixPreview.setFilters()`

How to read and change a dashboard's **filters** on the user's behalf — "show only Palgakulud", "switch to Divisjon 1", "add Tegelik to the budget card", "reset the filters" — without clicking the sidebar or a matrix's setup panel. A flat table's **column search** (the search row under its header — "show only the rows where Staatus is Avatud", "find 2024-1234") is section 7, `setSearch()`. A **double-click cross-filter** — switching it on for a row, combining, releasing — is section 8, `crossFilter()`. Companion to `references/table-data.md` (reading what the filtered tables then show).

**Never set filters through the DOM** (opening the Filtrid sidebar, picking in dropdowns, ticking the setup panel, clicking Set Filter). Clicks on the custom dropdowns and on filter-table rows land unreliably, a dropdown change without Set Filter does nothing visible, and a filter that narrows another must be picked first. The calls below put each value in through the same change handler the control uses, apply it through the same code as Set Filter / the setup panel's Apply, and **wait until the tables have reloaded** — read `data()` straight after.

---

## 1. Two kinds of filter

| Kind | Where the user sees it | Applies to | Call |
|---|---|---|---|
| **Page filters** | the *Filtrid* sidebar group (dropdown, input, toggle, btnSelect), and **filter tables** — a table whose selected rows are the filter's value (e.g. the employee/department tree on the left) | every table on the page, on **Set Filter** (a filter table applies on click) | `setFilters({ … })` |
| **Table filters** | a matrix's setup panel (⋯ → *Filters* / *Measures*): `location: 1` filters, the matrix's `defaultFilters`, and its **measure** toggle (`measureId`) | that one matrix — each matrix keeps its own values | `setFilters({ … }, { table })` |

The same filter can be a table filter on several matrices (on a budget page `yearTotal`, `formulas`, `accounts` and `measureId` sit on every card). Setting it with `{ table: 'finTablix' }` changes **only** `finTablix`; the other cards and the sidebar keep their values.

**Timeline filters** (`timeline_from`, `timeline_to`, `timeline_granularity`, `timeline_defaultPeriod`, `timeline_current`) are listed with `setBy: 'timeline'` and **refused** by `setFilters` — setting them underneath the bar would leave it showing another period. Set the period with `setPeriod()` (section 5).

---

## 2. Availability

- **A build that has it.** Probe `typeof window.tablixPreview?.setFilters === 'function'`. Missing means the deployed build predates it — say so; do not fall back to clicking.
- **Not admin-gated.** It sets what the user could set by hand.
- **On a `/grid/IndexPage?configId=<id>` dashboard.**

---

## 3. Reading — `filters()` and `filterOptions()`

```js
const p = window.tablixPreview;
const f = p.filters();               // page filters + every matrix's own
JSON.stringify({ unapplied: f.unapplied, page: f.filters.map(x => [x.filter, x.type, x.values, x.pending]), tables: Object.keys(f.tables || {}) });
p.filters('finTablix');              // one matrix's own filters: { table, filters: [...] }
```

Each filter:

| Key | Means |
|---|---|
| `filter` | its keyField — what to pass to `setFilters`. `id` and `name` are accepted too. |
| `type` | `dropdown`, `input`, `toggle`, `btnSelect`, `filterTable` (a table's selected rows), or `timeline`. |
| `values` | the **applied** values — what the tables are filtered by now. A dimension value comes as `{ value, caption }`. |
| `pending` | (page filters) a value changed but not applied yet — by you with `{ apply: false }` or by the user in the sidebar. `unapplied: true` at the top says some filter has one. |
| `multiple`, `required`, `hidden` | takes several values / cannot be cleared / not shown to the user (`visibility: 2`, e.g. a fixed `company`). |
| `options` | how many options its dimension has right now. |
| `narrows` | filters whose options this one restricts (`dimInfluence`) — set this one first. |
| `setBy: 'timeline'` | set by the timeline; refused by `setFilters`. |

At the top, `loading: true` says the page is still building itself (as in `info()`): the lists are incomplete - wait and call again.

The options a filter takes — what its dropdown or filter table offers **now** (a narrowing filter changes them):

```js
p.filterOptions('finhier', { search: 'palk' });          // { filter, multiple, total, options: [{ value, caption }], more }
p.filterOptions('measureId', { table: 'finTablix' });    // a matrix's measure list
```

`limit` (default 200) caps the list; `more` says how many were left out. Input and toggle filters have no options.

---

## 4. Setting — `setFilters()`

```js
const r = await p.setFilters({ finhier: ['Palgakulud', 'Tööjõumaksud'], department: 'Divisjon 1' });
JSON.stringify({ set: r.set, refused: r.refused, applied: r.applied, loaded: r.loaded });
```

- **Values:** an option's `value` or its `caption` (exact, then case-insensitive) — an array for a `multiple` filter; text for an input; `true`/`false` for a toggle; `null` or `[]` clears.
- **Order matters for narrowing filters.** Filters are set one after another in the order of the object's keys, each through its control's change handler, so a filter listed in another's `narrows` must come **after** it — its options are refreshed before it is resolved.
- **Applied by default** — as Set Filter: every table reloads and the call returns once they have drawn (`loaded: true`; `false` only after a 60 s timeout). Read `data()` right after it.
- **`{ apply: false }`** sets the sidebar but does not apply: the dropdowns show the new values and Set Filter lights up, for the user to press. Use it when the user wants to look before applying. `applyFilters()` applies later.
- **`{ table: '<matrixId>' }`** sets that matrix's own filters (measures, year total, show formulas/accounts…) and applies them at once, as the setup panel's Apply does. Only that matrix reloads.
- **Refusals don't stop the rest.** Each refused filter comes back in `refused` with its reason; the others are still set and applied.

| `refused[].reason` | Means |
|---|---|
| `not found - see tablixPreview.filters` | no filter with that keyField / id / name here — with `{ table }`, among that matrix's filters only |
| `is ambiguous - use its keyField` | the name matches more than one |
| `not an option now: …` | not in the filter's current options — maybe narrowed by another filter (`filterOptions`) |
| `takes one value` | several values for a filter that isn't `multiple` |
| `required - it cannot be cleared` | `null` / `[]` on a required filter |
| `a toggle takes true or false` / `not a number` / `contains < or >` | the value doesn't fit the control |
| `set by the timeline - use tablixPreview.setPeriod` | a timeline filter — section 5 |

Other calls:

```js
await p.applyFilters();                      // Set Filter, as things stand — including what the user changed by hand
await p.resetFilters();                      // Reset Filter: page filters back to their defaults
await p.resetFilters({ table: 'finTablix' }); // that matrix's own filters back to its defaults
```

`resetFilters()` also puts **filter tables** back to their default selection. The page's own Reset Filter button leaves them as they are — so after the user's button the department tree still shows their pick, after `resetFilters()` it doesn't. Say which one you did.

---

## 5. The period — `setPeriod()`

A timeline bar (the strip of months/years above the cards, with its period dropdown and Kuu / Aasta buttons) sets the page's period. `filters().timelines` says what it is set to:

```js
p.filters().timelines;   // [{ timeline, period, granularity, from, to, activeDate, periods, granularities, range, cells, exactDays? }]
```

`period` is the dropdown's named period, `'custom'` for a range picked on the bar; `periods` and `granularities` are what the bar offers now (a period limited to other granularities is left out); `range` the first and last day the bar covers; `cells` how its cells are labelled (`'01.22 … 12.29'`).

```js
await p.setPeriod({ period: '6 kuud' });                                   // the period dropdown
await p.setPeriod({ granularity: 'Aasta' });                               // the Kuu / Aasta buttons
await p.setPeriod({ granularity: 'Kuu', period: 'Jooksev aasta' });        // both
await p.setPeriod({ from: '2026-02', to: '2026-06' });                     // a custom range, as clicking cells
await p.setPeriod({ granularity: 'Aasta', from: '2025', to: '2026' });     // regroup, then the range
```

- **One call, as the controls do it.** Each option goes through the handler its control calls, the bar redraws, every table reloads, and the call returns once they have drawn, with the bar's new state in `timeline` (`done`, `loaded`, `timeline`).
- **A range follows the granularity.** `from` / `to` take a cell as the bar labels it (`'03.26'`, `'2026'`) or a date — `yyyy`, `yyyy-mm`, `mm.yyyy`, `yyyy-mm-dd`, `dd.mm.yyyy` — and are widened to **whole cells**: `from` to the start of its cell, `to` to the end of its cell, as clicking them selects. `{ from: '2026-02-05', to: '2026-03-28' }` sets 2026-02-01 … 2026-03-31 in Kuu, 2026-01-01 … 2026-12-31 in Aasta. Only a bar with its own from/to date pickers (`exactDays: true`) keeps single days. Tell the user the range you actually set when it was widened.
- **One end alone** keeps the other. `activeDate` moves the active-date marker (the last day of its cell, as a click); it is pulled inside the range if it falls outside.
- **The bar is shared, its setting is per card.** Each budget card keeps its own granularity and period and puts them back on the bar when it is shown; `setPeriod` sets the bar and the cards on screen, as the controls do.

| Refused with | Means |
|---|---|
| `"…" is outside the timeline (… … …)` | before or after the bar's `range` |
| `"…" is not a cell of the bar or a date (yyyy, yyyy-mm, yyyy-mm-dd)` | words like "last month" — work out the date yourself |
| `from (…) is after to (…)` | swapped ends |
| `no period "…" - see periods` / `"…" is not offered in …` | not in the dropdown, or not in that granularity |
| `no granularity "…" - this bar has …` | not one of the bar's buttons |
| `give a named period or from/to/activeDate, not both` | a named period has its own dates |
| `this timeline has no active date` / `this timeline selects one date - use activeDate` | the bar's own setup doesn't allow it |

A page with several timelines needs `{ timeline: <id> }`.

---

## 6. Other data without touching the page — `query()`

When the user asks something that needs data **for other filters than the ones on screen** — "look at last year's budget and fill this year's from it", "how did this business line do in 2024?" — do not change their filters or period to get it, and do not change them back afterwards. `query()` fetches a table's data with your own filters and leaves the page exactly as it is: same bar, same filters, same rows, nothing redrawn.

```js
const p = window.tablixPreview;
const now = p.data('flatTablix');                                            // what the user sees (2026)
const ly  = await p.query('flatTablix', { period: { from: '2025-01', to: '2025-12' } });
JSON.stringify({ done: ly.done, query: ly.query, refused: ly.refused, rows: ly.total, cols: ly.columns.length });
```

- **Same shape as `data()`** — `columns`, `rows` (with `keys` and `path`), `totals`, `rowLevels` / `columnLevels`, `offset` / `limit` paging. A matrix runs through the same mapping, conversion and row calculations as on screen, so subtotals, hierarchy and formula rows come out as the user would see them for those filters. Match last year's row to this year's by its `keys` (or `path`), and a column by its period: `'01.25 › Amount'` ↔ `'01.26 › Amount'`.
- **Overrides**, all optional; anything not given stays as the table loads now:
  - `period: { from, to, granularity? }` — as `setPeriod`'s range: a cell or a date, widened to whole cells of the granularity (`'Aasta'` → whole years). Both ends are needed.
  - `filters: { filter: value }` — page filters, as `setFilters` takes them.
  - `tableFilters: { filter: value }` — the table's own (measures, year total…), as `setFilters({ table })`. A measure such as **PY Amount** puts last year's figure on this year's columns directly, if that is what the user means — check the measure's meaning with them.
  - `selection: { table, at: [{ row, column }] }` — for a table that **follows another table's selection** (a matrix's detail tabs — *Muudatuste logi*, *Kommentaarid* — or any `updateOnSelect` target): the cells to send as if the user had clicked them, addressed like `stage()`. A row-triggered target takes `[{ row }]`. The cells are looked up in the source table loaded with the same `filters` and `period`, so you can ask about a cell the page isn't showing. Nothing gets selected and no tab opens. See *Who changed this cell* below.
- **A refused override fetches nothing** — `done: false` with `refused` (same reasons as `setFilters` / `setPeriod`). Data without the override would look like the answer to the question asked.
- **A result that changes the matrix's settings is refused too** (`done: false`, the server sent `gridMatrixSettings` for those filters): the page applies such a change by re-registering the table, which a copy must not do. Read it by setting the filters on the page instead, and say so.
- **Any table on the page**, also one on a tab the user hasn't opened (it loads only into the copy).
- **Then write with `stage()`** — putting this year's values in is `references/table-editing.md`; the user reviews and saves as always. Say which period and filters you read them from.
- A failed request shows the server's error toast to the user — the only thing a query can put on the screen.

### Who changed this cell — the change log through `query({ selection })`

"Who set this employee's Põhipalk for 01.27 to 8000?" is answered by the matrix's change-log tab (`log`, *Muudatuste logi*). **Don't read it with `data('log')`.** The tab is collapsed by default and a collapsed tab isn't reloaded when a cell is selected, so `data('log')` returns `[]` and looks like "no history". `select()` and real clicks have the same problem until the user opens the tab. Ask the log for the cell directly instead:

```js
const p = window.tablixPreview;
const r = await p.query('log', {
  filters: { department: '<employee>' },                                   // the employee, not the page's Divisjon 2
  selection: { table: 'finTablix', at: [{ row: ['Sisestusala', 'Põhipalk'], column: '01.27 › Eelarve' }] },
});
JSON.stringify({ done: r.done, refused: r.refused, log: r.rows.map(x => [x.updateDateTimeString, x.user, x.msg]) });
// [["25.09.2026 11:06","<user>","Amount: 0.00 -> 3,000.00"], ["21.10.2026 15:09","<user>","Amount: 3,000.00 -> 5,000.00"]]
```

- **One cell, one entered value.** The budget changelog API returns history only for a cell backed by exactly one stored value. It returns nothing for an aggregate (a department or division cell, where `cellData()` shows source rows with id `0|…`) or for several cells at once. A click in the UI gets the same nothing. So narrow `filters` down to the employee or row whose value was typed in. An empty result for an aggregate cell means "ask at the level where it was entered", not "no history".
- **Refused** (nothing fetched): `"log" does not send the selection of "…" - it follows …` (wrong source table; the message lists the ones it follows), `row not found` / `column not found` / `… is ambiguous` (address it as in `stage()`), `the cell is empty - selecting it sends nothing`, `give the cells or rows to send`, `… selects one row at a time (rowSelectMultiple is off) …`, and `… loads no data of its own (sampleData, no dataUrl) …` (a source without a `dataUrl` can't be reloaded under the query's filters).
- The given cells take the place of whatever the page has selected in that source table. Selections the target sends from *other* tables are left as the page has them.

**Which selections `selection` covers.** It follows the target's `includeTableSelections` entry for that source, the way a click does:

| The target follows | Give | Sent as |
|---|---|---|
| a matrix's **cells** (detail tabs, `updateOnSelect` with `trigger: 'cell'`) | `[{ row, column }]` | body `selection` + `dataSelection` (the cells' source rows), `includedCols` applied |
| a **flat row** whose id goes into the target's **url** (`[[selection0]]`, `selectionReplacements`) | `[{ row }]` — id or `{ field: value }` | the url token, from the given rows (`deliminator` joins several, `type: 'count'` counts them) |
| a **flat row** sent in the body | `[{ row }]` | body `selection` |
| a **matrix row** (`trigger: 'row'`) | `[{ row }]` | body `selection` |
| a **flat cell** (`trigger: 'cell'` on a flat source) | `[{ row, column }]` | body `selection` as `{ rowId, column: value }` |
| a **cross-filter** (`selectionType: 'cross-filter'`) | as its trigger | the same as a selection; sent even when no cross-filter is active on the page |

- **Several rows of a single-select table are refused** (`… selects one row at a time (rowSelectMultiple is off) - query each row separately`). A click can't select two, and the target's API isn't built for it.
- **Not covered:** a target with **`dataOrigin: 1`**, which doesn't fetch at all but shows the source's rows behind the selection. `query` on it would fetch its own url, or throw "no dataUrl". Read the source with `cellData()` or `data()` instead. Several cross-filtering tables combined with Ctrl can't be expressed either: one `selection` names one source, and the others come from the page. Forms that follow a selection aren't tables: use `select()` + `openForm()`.
- Why a click on the page isn't enough: a target in a collapsed container counts as not visible, so selecting doesn't reload it. That applies to every selection-driven table, not just the log.
- **Builds:** `query` takes `selection` from the 2026-10-21 build on. An older build ignores the key and returns the log of the page's own selection, which is usually empty. Check that `r.query.selection` is present before trusting an empty answer.

---

## 7. Column search of a table — `search()` / `setSearch()` / `clearSearch()`

The row of search boxes under a table's header (toolbar funnel *Show search*) narrows the rows **client-side**, per column — "show only Avatud orders", "find the row with 2024-1234", "amounts over 1000". It is not a filter in the sense of sections 1–6: nothing is requested from the server, the page filters stay as they are, and the user clears it with the toolbar's *Clear search*. **Never type into the search boxes or pick in their dropdowns** — set it in one call:

```js
const p = window.tablixPreview;
p.search('flatTablix');                                            // { table, searchable, open, columns: [{ field, header, formatType, dropdown?, value? }], shown, total }
const r = await p.setSearch('flatTablix', { status: 'Avatud', amount: '>=1000' });
JSON.stringify({ set: r.set, refused: r.refused, shown: r.shown, total: r.total });   // then read data()
await p.setSearch('flatTablix', { amount: null });                 // clears that column, keeps the others
await p.clearSearch('flatTablix');                                 // Clear search: every column
```

- **Builds:** probe `typeof window.tablixPreview?.setSearch === 'function'`; missing means the deployed build predates it — say so, do not fall back to typing. Not admin-gated.
- **Which tables:** a flat table with a search row (`Table.isSearchable`); `search()` says `searchable: false` and `setSearch` throws otherwise. A matrix in stepped layout with `searchDropdown` row columns has the same row; verified on flat tables only.
- **Columns** by `field` (valueField) or a unique header name, among the columns `search().columns` lists — the searchable columns the user has on screen, in their order. A column the user has hidden, or one with `isSearchable: false`, is refused.
- **Value = what the user would type.** Plain text matches as a substring, case-insensitive, against the cell as shown (a dimension column by its caption, a number as formatted). The row's operators work as typed: `>=1000`, `<0`, `100..200` (range), `!100..200` (outside), `!text` (not containing), `!""` (not empty), `text*` / `*text` / `a*b` (starts / ends / starts and ends with), `!text*`. **Several terms** match either: `'Avatud, Ootel'` — an array `['Avatud', 'Ootel']` is joined the way the search dropdown joins picked options (with `; ` on a number column with decimals, since `,` is its decimal mark there). A bool / btnSelect column takes the option's id (`'true'`, `1`, or the custom value), as its buttons do. `null`, `''` or `[]` clears the column.
- **Columns not named keep their search** — `setSearch` merges. `clearSearch()` is the toolbar's *Clear search*.
- **Unsaved changes block it.** Like the search row itself ("Please save the changes before filtering"), `setSearch` throws while the table holds unsaved edits — the user saves, or `discard()`, first.
- **The search row is opened** (as the funnel button does) so the user sees what narrows the table; `{ open: false }` leaves it as it is — the search applies either way, since the state lives in the table's setup, not in the boxes.
- **What the user sees:** each searched column gets a thin muted underline on its header and a small funnel in the header label, right after the text and its ⓘ (hover shows the searched text — so a packed search row still tells what is searched), and the *Clear search* button carries a count of searched columns. The same marks appear when the user types, so what you set looks like what they would have done.
- **Then `data()`** returns only the rows the search leaves (`total` is the shown count) and carries `searched: { field: text }` while a search is active — section 4 of `references/table-data.md`. Totals follow the search (footer; group totals only with `rowGroupTotalsFiltered`).

| Refused / thrown | Means |
|---|---|
| `not found - see tablixPreview.search` | no column with that field or header; `search().columns` lists them |
| `is ambiguous - use its field` | the header name matches more than one column |
| `hidden by the user - a hidden column is not searched` | the user hid the column (bookmark / setup); a search on it would not apply |
| `not searchable (isSearchable is false on the column)` | the column has no search box |
| `takes text, a number, true/false or an array of them` | an object was given |
| throws `has no search row (isSearchable is false)` | the table has no search row at all |
| throws `has unsaved changes - the user saves or discards them first` | pending edits on the table |

Report it the way the page shows it: "Filtered Staatus to *Avatud* — 12 of 84 rows; Clear search (the funnel-x on the card) or ask me to clear it." A search that leaves 0 rows is usually real — the text is not in that column as shown (a dimension column is searched by caption, not id) — say so rather than retrying blindly; `data()` without the search, or `search().columns`, shows what is there.

---

## 8. Cross-filters — `crossFilters()` / `crossFilter()` / `releaseCrossFilters()`

A flat table whose `updateOnSelect` has a target with `selectionType: 'cross-filter'` filters that target when the user **double-clicks a row**. The first click selects it, which updates the ordinary `selection` targets. The second click, on the already selected row, switches the cross-filter on: the row gets the `cross-filter-row` mark, the card gets `active-cross-filter`, and the targets reload with the row in their `selection`. Clicking the row again releases it. **Ctrl+double-click** adds the cross-filter to those already active instead of replacing them. **Never double-click through the DOM.** It depends on a 500 ms debounce and an edit-mode guard, and on an `inlineEdit` table a double-click on a cell opens the editor instead. Use these calls:

```js
const p = window.tablixPreview;
p.crossFilters();                                   // [{ table, name, active, rows, targets: [{ element, type, shown }] }]
await p.crossFilter('orders', 2);                   // row by id or { field: value }: selects it, cross-filters, waits for the targets to reload
await p.crossFilter('customers', 10, { combine: true });  // Ctrl+double-click: keeps the active ones
await p.crossFilter('orders', null);                // release, as clicking the row again does
await p.releaseCrossFilters();                      // release every cross-filter on the page
```

- **Returns** `{ applied, row, active: [tables], already?, notReloaded? }`. `already: true` means that row was already cross-filtering, and nothing reloaded. `notReloaded` lists targets that aren't on screen (collapsed container, a tab behind another). Those aren't reloaded now, exactly as with a click, so tell the user they'll see it when they open that tab.
- **Without `combine`** every other cross-filter is released first, as a plain double-click does. Switching to another row of the same table releases the old one and filters on the new one.
- **Releasing one releases all.** That's how the page's release click works, and `crossFilter(table, null)` does the same: `releasedTables` says which went off. Targets reload unfiltered. The row stays *selected* (ordinary `selection` targets keep it); only the cross-filter goes.
- **Always release what you set when the user is done with it**, or when your answer no longer needs it. A cross-filter changes what every target shows until it's released, and it isn't saved anywhere: a reload clears it too. Say it's on ("Detail is filtered to order A-2; ask me, or click the row again, to release it"), and release it before setting up a different view.
- **Refused:** `"…" cross-filters nothing (no updateOnSelect with selectionType "cross-filter")`, `row not found in "…"`, `reference matches more than one row`, `no cross filter is on for "…"` (a release with nothing on), and `cross filters are set from a flat table's rows` (a matrix). The selection itself can be refused too, e.g. by an unsaved-changes prompt.
- **To read a target as if cross-filtered without changing the view**, use `query(target, { selection: { table, at: [{ row }] } })` instead (§ 6).
- **Builds:** probe `typeof window.tablixPreview?.crossFilter === 'function'`.

---

## 9. Report back

Filters change what the user sees immediately — tell them in one line what is filtered now (with captions, not ids; the period as the months it covers) and how to undo it ("Reset Filter, or ask me"). When a filter came back refused, say why and offer what `filterOptions()` lists. An empty matrix after filtering is usually real — that combination has no data — not a failed call; check `values` in `filters()` and say so.

