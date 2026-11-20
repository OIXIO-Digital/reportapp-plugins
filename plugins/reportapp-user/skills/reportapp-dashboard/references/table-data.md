# Reading a table's data — `tablixPreview.data()`

How to get the rows a dashboard table or matrix is showing, as JSON, so you can answer questions about the data itself: totals, comparisons, outliers, "which account is over budget", "what does this matrix say about Q3". 

Use it whenever the user asks about what is on screen rather than how the screen is built — the dashboard menu's "Alusta analüüsi" action starts here. Do not scrape the DOM for this: the grid draws rows in chunks of 100 as the user scrolls, so the DOM holds only part of a large table, and a matrix's cells carry formatted text (`7 250`), not values.

---

## 1. Availability

- **Any signed-in user.** `data()` is **not** admin-gated. It returns only rows the page already fetched for this user — data they can see and export anyway.
- **A `/grid/IndexPage?configId=<id>` dashboard tab.** The namespace also exists on `/masterdata`, but there are no dashboard tables to read there.
- **A build that has it.** Probe `typeof window.tablixPreview?.data === 'function'`. If it is missing, the deployed build predates the method; say so rather than falling back to DOM scraping.
- **The table has loaded.** A matrix in an inactive tab usually has 0 rows until the user opens that tab (the tab triggers the fetch). A `total: 0` on a tab you have not opened is not an empty dataset — open the tab, or ask the user to, and read again.

---

## 2. Finding the table id

The id is the key in `window.customTables` — a table's `id`, a matrix's `valueField`. Ask for a table that does not exist and the error lists the real ones:

```js
try { window.tablixPreview.data('?'); } catch (e) { e.message; }
// "tablixPreview: no table "?". Available: comments, log, thread, finTablix, finTablix1, finTablix2"
```

To match an id to what the user sees, pair each with its name and type:

```js
JSON.stringify(Object.entries(window.customTables)
  .filter(([k, t]) => k !== 'cellNote' && t.currentSetup?.containerId && !t.isFilter && !t.isDropdown)
  .map(([k, t]) => ({ id: k, name: t.name, type: t.type, rows: (t.data || []).length })));
```

A matrix's `name` is its tab label (`Esimene`, `Teine`, …); several matrices in one card are tabs, each its own id.

---

## 3. Output safety — compute in the browser, return the answer

Tool output is silently truncated at roughly 4 KB. A 230-row matrix with 29 columns is far larger. **Never return `data()` whole.** Either:

- **Compute in the browser and return the result** — the preferred way. Sums, maxima, filtered lists and comparisons all run in the same call; return a few numbers or a short list.
- **Page it** with `offset` / `limit` when you genuinely need rows, and keep each page small.

```js
const d = window.tablixPreview.data('finTablix');
JSON.stringify({ total: d.total, columns: d.columns.map(c => c.label).slice(0, 30) });   // shape first
```

---

## 4. Call

```js
window.tablixPreview.data(tableId, { visibleOnly, offset, limit })
```

| Option | Default | Effect |
|---|---|---|
| `visibleOnly` | `false` | Drop columns the user has hidden |
| `offset` | `0` | First row to return |
| `limit` | all | Maximum rows to return |

Every call returns a fresh copy; mutating it changes nothing on the page. `total` is always the full row count, so `offset + returned < total` means more pages.

What the rows are, for both shapes:

- **What the user sees, in the user's order.** If a column search is active, only matching rows; if the user sorted, rows come back sorted. Verified: search "an" + sort by Kaal gave the same 9 rows in the same order as the grid.
- **`searched`** (`{ field: text }`, absent when nothing is searched) says which column search is narrowing the rows — so `total` smaller than the table is read as a search, not as missing data. Reading and setting the search without typing: `tablixPreview.search()` / `setSearch()`, `references/filters.md` section 7.
- **All of them**, not just the rows drawn so far.
- **Raw values.** Numbers unformatted (`7250.1234567`, not `7 250`), dates as stored (`"2026-01-01"`, months as `20260201`), percentages as fractions (`0.38` for 38 %).
- **Dimension columns hold ids** in flat tables — see section 7.
- **Unsaved changes included.** Values the user typed or `stage()` put in but nobody has saved yet are what `data()` returns, and `pending` lists them (`{ rowId: [fields] }`, absent when nothing is pending). Matrix totals and formula rows are the exception — they only recalculate on Save (`references/table-editing.md` section 8).

To **write** values back — fill a budget, change cells for the user to accept with Save — see `references/table-editing.md`.

---

## 5. Flat tables

```js
{ tableId, name, type: 'flat', rowId, total, offset, returned, pending,
  columns: [{ field, header, visible, formatType, dimId, editType, isRequired, editable }],
  cellOverrides: { 96671: { number: { editable: false } } },
  rows:    [{ id: 96671, text: 'Apple', number: 39.19, dropdown: 96225, ... }] }
```

- `editable` / `editType` / `isRequired` on a column, and `cellOverrides` (the cells whose editable, edit type, format or dimension differ from their column because of `customEditable`, `customFormats` or a cell condition), are for writing back with `stage()` — `references/table-editing.md` section 3. `cellOverrides` is absent when no cell differs.

- Rows are keyed by `field` (`valueField`); `columns[].header` is what the user sees. Talk to the user in headers, not fields.
- Only configured columns (plus the row id and, on hierarchical tables, the depth/parent fields) are included — internal row state is left out.
- `formatType` says how to read a value: `number`/`int`, `percentage` (fraction), `date`, `month` (`yyyymmdd`), `bool` / `boolInt`, `dimText` / `dropdownMulti` / `btnSelect` / `dimIcon` (dimension ids), `json` (a JSON string), `richText` (HTML).
- Synthetic row-group header rows are dropped; grouped tables return their real rows only.

---

## 6. Matrices

A matrix's internal rows are keyed by dimension codes joined with an invisible separator — meaningless to a reader. `data()` decodes them:

```js
{ tableId, name, type: 'matrix', total, offset, returned, pending,
  rowLevels:    ['Tase 1', 'Tase 2', 'Tase 3'],
  columnLevels: ['Periood', 'Mõõdik'],
  columns: [{ label: '01.26 › Eelarve',
              path: { Periood: '01.26', Mõõdik: 'Eelarve' },
              keys: { period: '2026-01-01', measureSeq: '4019' },
              isSubtotal: false, visible: true, formatType: 'number',
              code: '2026-01-01\x1F4019\x1F' }],
  rows: [{ id: '3243\x1F3249\x1F3250\x1F',
           label: 'Sotsiaalmaks', level: 2, levelName: 'Tase 3',
           path: ['Lõplikud eelarve numbrid', 'Tööjõumaksud', 'Sotsiaalmaks'],
           keys: { accountGroup1Id: '3243', accountGroup2Id: '3249', accountGroup3Id: '3250' },
           values: { '01.26 › Eelarve': 7250.1234567, '02.26 › Eelarve': 4100.4567891, ... },
           editable: ['01.26 › Eelarve', ...] }],
  totals: null }
```

Row `id` and column `code` are the raw grid keys; row `editable` lists the column labels this user may change in that row, and is absent when there are none. All three are for writing back with `stage()` — `references/table-editing.md`.

**Columns.** `label` joins the caption of every column level, outermost first, so it follows the matrix's own column order — `Tegelik › 01.26` on a measure-first matrix, `01.26 › Eelarve` on a period-first one. The label always names the measure, even when the grid's header hides it (a single-measure matrix shows only `01.26`). Use `path` to filter by one level (`c.path.Mõõdik === 'Eelarve'`), `keys` for the raw codes. Labels are unique; `row.values` is keyed by them.

**Rows (hierarchical layout).** `label` is the row's caption, `path` the captions from the top down to this row, `level` its indentation depth (0 = top). `levelName` is **which row dimension the row is** — read it, not `level`, to know what a row represents. `keys` are the raw dimension ids per level, and a `'0'` means *all of that level*.

The two differ when the last row level is shown as a breakdown under every level. On a matrix with rows `Tase 1 › Tase 2 › Tase 3 › Töötaja`:

| `path` | `level` | `levelName` | `keys` | Means |
|---|---|---|---|---|
| `Lõplikud eelarve numbrid` | 0 | Tase 1 | `3243` | the whole group |
| `Lõplikud eelarve numbrid / 230` | 1 | **Töötaja** | `3243/0/0/143` | employee 230, over all accounts of the group |
| `… / Tööjõumaksud` | 1 | Tase 2 | `3243/3249` | one sub-group |
| `… / Tööjõumaksud / Sotsiaalmaks / 230` | 3 | Töötaja | `3243/3249/3250/143` | employee 230 on one account |

**Rows (stepped layout)** — the setup's *Hierarhiline* switch off, one column per row dimension. There is no `label`/`level`/`path`; the dimension captions are in `fields`, keyed by row-level header: `fields: { 'Tase 1': 'Lõplikud eelarve numbrid', 'Tase 2': null, 'Tase 3': null, 'Töötaja': '230' }`. `null` is a level the row does not have (the grid shows it blank); `keys` still has the ids.

**Reading the numbers correctly:**

- **Rows overlap — never sum all of them.** Parent rows are subtotals, and breakdown rows (the Töötaja rows above) repeat the same money at every level. "Rows with no children" is not a safe rule either: `…/230` under the top row has no children and is still a subtotal. Sum one **slice** — rows with the same parent path and the same `levelName` — or read the single row that is the scope you want. A parent row can also be blank (`null` everywhere) while its breakdown rows carry the numbers; that is the matrix's layout, not missing data.
- **Empty cells are `null`**, not 0 — nothing is booked there. Say "no value" rather than "zero".
- **Totals and formula measures are ordinary columns.** Year totals (setup *Aasta kokku*) come back as e.g. `Tegelik › 2026`, or `Tegelik › Total › 2026` when Aasta is a column level too; rolling ranges as `09.25..09.26 › Eelarve`. With *Näita valemeid* / *Näita kontosid* on, the formula measures appear as `Valem › Valemid`, `Muutuja › Valemid`, `Konto › Valemid`, and their values are **strings** (`"(P+Li+Pu) * 33 / 100"`, `"So"`, `"4723"`). Exclude totals and formula columns before aggregating across periods.
- **Row captions are the grid's own.** Where the grid shows a code (e.g. employee `230`), so does `label`.
- `totals` is the grand-total row, keyed like `values`, or `null` when the matrix has none.

### Source rows behind one cell — `cellData()`

`data()` gives the aggregated grid. To see **which source records one matrix cell is built from** ("why is this 29 500?", "which accounts are in here?"), have the user click the cell, then:

```js
window.tablixPreview.cellData()            // the focused matrix cell
window.tablixPreview.cellData('finTablix') // only look in this matrix
```

```js
{ tableId: 'finTablix', rowId: '3243\x1F', colId: '2026-01-01\x1F4019\x1F',
  value: 29500.12345678, count: 9,
  data: [{ id: 'C|20260101|1001|0|1', quantity: 5000.00, accountCode: '4000',
           measureName: 'Eelarve', monthYear: '01.26', accountGroup3: 'Põhipalk', ... }, ...] }
```

- **It reads the focused cell** (`td.focus`, set by a click). With several matrices on the page, the one holding keyboard focus wins, else the first focused matrix cell. Flat-table cells are ignored. If nothing is focused it returns `null` and warns `no focused matrix cell - click a cell first` — ask the user to click the cell; don't try to guess which one they mean.
- **`data` holds raw source rows** with every field the data source returns, not only the configured columns. Field names are the source's own (`quantity`, `accountGroup1Id`, …), not headers. `rowId` / `colId` are the internal separator-joined codes; use `data()` for readable labels.
- **Same output safety as section 3.** A top-level cell can hold hundreds of rows — return `count` and a few picked fields, not `data` whole.
- **`sum(data)` need not equal `value`.** The source can include rows for the group lines themselves as well as their children: on `finTablix` the Tase 1 cell `01.26 › Eelarve` returned 9 rows whose `quantity` summed to exactly 3× `value` (line 3243, sub-group 3244 and their accounts). Don't report the plain sum as the cell's value. Filter to the leaf lines, or just compare against `value`.
- Not admin-gated, same as `data()`.

```js
const c = window.tablixPreview.cellData();
JSON.stringify(c && { value: c.value, count: c.count,
  rows: c.data.map(r => [r.lineId, r.accountCode, r.accountGroup3, r.quantity]) });
```

---

## 7. Dimension ids in flat tables

A flat table's dimension columns (`dimId` set in `columns`) carry the id, not the caption. Resolve them through the loaded dimension's `result` rows, whose caption field is named by the dimension's `displayedValue`. **Do not use `resultDict`** — it can be empty even when `result` is loaded.

```js
const d = window.tablixPreview.data('SisestusTest', { limit: 5 });
const dims = window.dimensions;
const captionsFor = dimId => {
  const dim = Array.isArray(dims) ? dims.find(x => x.id === dimId) : dims[dimId];
  const field = dim?.displayedValue || 'caption';
  const map = {};
  (dim?.result || []).forEach(x => { map[x.id] = x[field]; });
  return map;
};
const status = captionsFor(d.columns.find(c => c.field === 'dropdown').dimId);
const cats = captionsFor(d.columns.find(c => c.field === 'dropdownMulti').dimId);
JSON.stringify(d.rows.map(r => ({
  text: r.text,
  status: status[r.dropdown] ?? r.dropdown,
  // multi-selects are comma-separated id strings
  category: String(r.dropdownMulti ?? '').split(',').filter(Boolean).map(id => cats[id.trim()] ?? id),
})));
// [{"text":"Apple","status":"Done","category":["puuvili"]}, {"text":"Arugula",...,"category":["lehtköögivili","ristõieline köögivili"]}, ...]
```

A dimension with `result` of length 0 has not loaded (or its API failed). Some `formatType`s store a caption directly (`inputdropdown`, `btnSelect` often hold text such as `"Yes"` or `"kg"`); if a value is not found in the map, show it as-is.

---

## 8. Recipes

**Largest values in one column (flat):**

```js
const d = window.tablixPreview.data('SisestusTest');
JSON.stringify(d.rows.slice().sort((a, b) => (b.number ?? 0) - (a.number ?? 0)).slice(0, 5).map(r => [r.text, r.number]));
```

**Per-measure, per-month total of a group, from one slice (the employee rows directly under it), excluding totals and formula columns:**

```js
const d = window.tablixPreview.data('finTablix1');
const top = d.rows.find(r => r.level === 0);
const slice = d.rows.filter(r => r.path.length === 2 && r.path[0] === top.label && r.levelName === 'Töötaja');
const monthCols = d.columns.filter(c => /^\d\d\.\d\d$/.test(c.path.Periood));
const out = {};
slice.forEach(r => monthCols.forEach(c => {
  const v = r.values[c.label];
  if (typeof v === 'number') out[c.label] = (out[c.label] || 0) + v;
}));
JSON.stringify({ rows: slice.length, eelarve0126: out['Eelarve › 01.26'] });
// {"rows":9,"eelarve0126":29500.12345678} — equal to the group's own cell on the single-measure tab
```

Cross-check a derived total against a cell the matrix shows directly whenever one exists; it catches a wrong slice immediately.

**One account across all periods, found by caption path:**

```js
const d = window.tablixPreview.data('finTablix');
const row = d.rows.find(r => r.path?.join(' / ') === 'Lõplikud eelarve numbrid / Tööjõumaksud / Sotsiaalmaks');
JSON.stringify(row && Object.entries(row.values).filter(([, v]) => v != null));
```

Name the table, the filter/search state and which columns you summed when you report a number, so the user can match it against the screen.

---

## 9. Troubleshooting

| Symptom | Cause |
|---|---|
| `tablixPreview.data is not a function` | Build predates the method. |
| `tablixPreview.cellData is not a function` | Build predates `cellData()` (newer than `data()`). |
| `cellData()` returns `null`, "no focused matrix cell" | No matrix cell has been clicked, or the `tableId` passed is not the matrix holding the focus. |
| `cellData()` returns `null`, "no matrix entry for row …" | The focused cell is a header/caption cell, or the matrix was redrawn since the click. Click a value cell again. |
| Throws `no table "…"` | Wrong id; the message lists the real ones (section 2). |
| `total: 0` on a matrix | Its tab has not been opened yet (section 1), or the selection really has no data. |
| Fewer rows than you expected | A column search is active — `data()` follows it and says so in `searched`. Check with the user before assuming data is missing; `clearSearch(tableId)` clears it (`references/filters.md` section 7). |
| A number does not match the screen | Formatting (the grid rounds), a parent row being summed with its children, or a total/formula column included (section 6). |
| Probe returns truncated JSON | You returned rows; compute in the browser instead (section 3). |

---

## 11. What kind of page this is, and a budget's FRD — `info()` / `frd()`

Not every dashboard is a plain Tablix view. A **budgeting solution** (metadata root `type: "budget"`) shows budget cards whose matrices are built on an FRD row definition: each such matrix carries an `frdCode` in `gridMatrices`. Call `info()` first on an unfamiliar page:

```js
const i = window.tablixPreview.info();
// { configId: 370, type: 'budget', isBudget: true,
//   tables: [{ tableId: 'finTablix', name: 'Esimene', type: 'matrix', frdCode: 'F1' }, ...], frd: '...' }
```

**`loading: true` means the page is still building itself** - a heavy dashboard prepares its matrices and timeline for several seconds after navigation, and until then `tables` (and `filters()`) are incomplete: on a page with five matrices and a timeline, a call 3 s after load listed two flat tables and nothing else. Wait and call again; never conclude "this page has no matrix / timeline" from a result that carries `loading`.

**Read-only.** On a budget page the FRD is information only. `frd()` has no write counterpart, and nothing else may be used to change the FRD from a budget page: no `stage()`, forms or imports aimed at it, and no direct calls to the finance API. If the user wants a line, formula or account changed, tell them it is done in the FRD editor. Entering budget values on the card itself (`stage()`, the user saves) is separate and unaffected.

A budget card shows only the measures the user has switched on, so a row's formula, budget account or account range may not be on screen at all. `frd()` returns the **whole** row definition from the finance reporter API, whatever is switched on. It is async, read-only, and not admin-gated (the API applies its own permissions to the user):

```js
const f = await window.tablixPreview.frd('finTablix'); // a matrix's tableId, an FRD code, or nothing when the page has one frdCode
// { frdCode, id, name, active, lineCount, lines: [
//   { id, parentId, lineNum, level, path: ['TULUD'], name: 'Müügitulu', type: 'Kanne', typeId: 0,
//     variable: 'MT', budgetAcct: '3000', formula, formulaHistory, calc: [{ line, lineId, variable, formulaType, calc }],
//     filters: { Konto: ['3000..3099', 'not 3050'] }, totalType, transactionType, reverseSign, isPositive, format, settings } ] }
```

- `formula` is the newest one; `formulaHistory` appears only when a line has several by start date. Formulas reference other lines by `variable`.
- `calc` lists the lines a Sum / Minus / Multiply / Divide / Formula line is built from, by name and id.
- `filters` are the account (and dimension) ranges a posting line collects, keyed by the constraint's name in the user's language; excluded ranges start with `not `.
- Empty fields are left out. The result is cached per code for the page's life.
- Match FRD lines to matrix rows by name (`lines[].name` ↔ `data().rows[].label`, `path` ↔ `path`).
- Same output rule as `data()` (section 3): find the lines you need in the browser and return those, not the whole definition.

| Symptom | Cause |
|---|---|
| `frd is not a function` | Build predates the method. |
| `several FRDs on this page` | Pass the matrix's `tableId` (from `info()`) or the code. |
| `the finance reporter API address is not on this page` | The server build is too old. |
| `no budget FRD "…"` while the card shows rows | The budget data comes from another environment than the finance API. |
| `401` / `403` | The finance API refused this user. Say so; don't work around it. |

