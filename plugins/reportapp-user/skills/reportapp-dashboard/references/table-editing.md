# Writing values into a table — `tablixPreview.stage()`

How to put numbers or values into a dashboard matrix or flat table on the user's behalf — "fill in the budget for employee 146", "set every Autokulud month to 180", "mark these three rows Finished" — so that **the user accepts them with the table's own Save button**. Companion to `references/table-data.md`, which covers reading; this file covers writing.

**Never type into the grid through the DOM for this** (clicking cells, keyboard input, synthetic paste). It is slow, it silently skips locked cells, and it can commit on every keystroke. `stage()` writes a whole batch in one call (216 matrix cells in 18 ms), checks every cell the way the grid does, and holds the result for the user.

---

## 1. The rule: stage, never save

`stage()` marks cells as changed exactly as if the user had typed them — green outline, Save button enabled — and **sends nothing to the server**. Committing is the user's action:

- **Save** (💾 in the table's toolbar) sends the staged changes through the table's normal save.
- **Reset**, **Close edit** (✕ on a `lockedEdit` table) or `tablixPreview.discard(tableId)` throws them away. `discard()` also removes unsaved **new** rows (`stageRows()`, or the grid's own "+ New"), which Reset leaves behind as blank rows.

Do not click Save yourself, and do not call anything that saves. Stage, report what you staged in plain language, and tell the user to review and press Save.

---

## 2. Availability

- **A build that has it.** Probe `typeof window.tablixPreview?.stage === 'function'`. Missing means the deployed build predates it — say so; do not fall back to DOM typing.
- **A user with edit rights.** Not admin-gated, but it only writes cells the signed-in user could edit by hand (`canEdit`, the table's `inlineEdit`, per-cell locks). For a read-only user every edit comes back refused.
- **Matrices and flat tables.** Card-tables and graphs are not covered; forms have their own calls — `references/forms.md`.
- **Edit mode is handled.** On a `lockedEdit` table (the usual flat masterdata table, which needs its Edit button pressed first) `stage()` switches edit mode on itself, exactly as the Edit button does. If the user has no Edit button there, every edit is refused with `table is not in edit mode and this user has no Edit button`.

---

## 3. Workflow

```js
const p = window.tablixPreview;
// 1. read what is there and what may change
const d = p.data('finTablix');
// 2. for dropdown / multi-select / btnSelect cells, look up the allowed values
const opts = p.options('SisestusTest', 96677, 'dropdown');
// 3. stage everything in ONE call
const res = await p.stage('finTablix', edits);
// 4. report counts and refusals only - never return data() whole (table-data.md section 3)
JSON.stringify({ staged: res.staged.length, unchanged: res.unchanged.length, refused: res.refused });
```

Build the whole edit list first and stage it in one call. One call is one undo step for the user, one validation pass, and one set of highlighted cells to review.

### Which cells may change — read it before building edits

- **Matrix:** each row in `data().rows` has `editable` — the column labels this user may change in that row. No `editable` key means nothing in that row can change (a subtotal of a locked level, a calculated row). Only build edits for labels in that list.
- **Flat table:** each `data().columns[]` entry has `editable`, `editType` and `isRequired`. A row can override a column — `customEditable`, `customFormats` and cell conditions can make one cell locked, change its edit type, its format or its dimension — and `data().cellOverrides` lists exactly those cells: `{ rowId: { field: { editable: false } } }`. A cell is editable when its column is, unless `cellOverrides` says otherwise.

`stage()` re-checks every cell anyway and refuses the ones that are not editable, so reading first is about building the right list, not about safety.

---

## 4. Addressing rows and columns

Each edit is `{ row, column, value }`, using what `data()` returns.

**Matrix**

| `row` | Example |
|---|---|
| its `id` (exact, always unique) | `'3252\x1F3258\x1F6949\x1F'` |
| its `path` — array of captions, or joined with `' › '` | `['Sisestusala', 'Muud kulud', 'Autokulud']` |
| its `label`, **only if unique** | `'Koormus'` |
| its `keys` object | `{ accountGroup1Id: '3252', accountGroup2Id: '3258', accountGroup3Id: '6949' }` |

| `column` | Example |
|---|---|
| its `label` | `'01.26 › Eelarve'` |
| its `code` (exact) | `'2026-01-01\x1F4019\x1F'` |
| its `keys` object | `{ period: '2026-02-01', measureSeq: '4019' }` |

Labels repeat more often than you expect — one budget matrix has two rows called `Põhipalk`. A non-unique label is refused (`row reference matches more than one row`), never guessed; use the path or id.

**Flat table**

| `row` | Example |
|---|---|
| the row id (`data().rowId` names the field) | `96677` |
| an object of field values matching exactly one row | `{ text: 'Banana' }` |

`column` is the `field` (`'dropdown'`) or a unique `header` (`'Staatus'`).

Rows are matched against every row of the table, not just the rows a search leaves visible.

---

## 5. Values

Give values the way a user would type them. `stage()` converts them into what the cell stores, with the same conversion the grid's own paste uses, and refuses what does not fit:

| Cell | Give | Stored |
|---|---|---|
| number | `1234.56`, `'1 234,56'`, `'12.5'` | number |
| int | `42`, `'42'` | integer |
| percentage | **as shown**: `38` or `'38%'` | fraction `0.38` |
| bool / boolInt / custom bool | `true`/`false`, `'true'`, `1`/`0` | `true` / `1` / the custom value |
| date | `'2027-03-15'` or `'15.03.2027'` | `'2027-03-15'` |
| month | `'2027-05'`, `'05.2027'` or the stored `20270501` | `20270501` |
| dropdown / dimIcon / btnSelect | the option's `value` (id) **or** its `label` from `options()` | the id |
| multi-select | an array, or a `','`-joined string, of ids or labels | `'96405,96406'` |
| input-dropdown | any text (suggestions via `options()`, `freeText: true`) | the text |
| color | `'#8b5a2b'` | as given |
| json | an object, or a JSON string | the JSON string |
| rich text | HTML — sanitised like the editor's own output | sanitised HTML |
| text / textarea | plain text | as given |
| any | `null` or `''` | cleared, as the Delete key does |

- **`<` and `>` in text** (text, textarea, input-dropdown, json) are taken since the 2026-10-20 build unless the column has `allowAngleBrackets: false`, which refuses them with `contains < or > - markup is not allowed in a text cell`. **Older builds refuse them on every such cell**: text cells render as HTML and the grid kept markup out by refusing those keys when typed, and `stage()` mirrored that without knowing what the text means — an FRD formula with a comparison (`a > b ? c : d`) was refused too. On a refusal hand the text to the user. Rich text is the one place HTML is allowed, and it goes through the same sanitiser as the rich-text editor (`<script>`, `on…` handlers and disallowed tags are dropped).
- `tablixPreview.options(tableId, row, column)` returns `{ dimId, editable, multiple, freeText, options: [{ value, label }] }` for a dimension cell, or `null` for a cell without one. It uses that row's own dimension — `customEditable` can point one row's cell at a different dimension than its column.

---

## 6. The result

```js
{ staged:    [{ row, column, value }],   // now pending, waiting for Save
  unchanged: [{ row, column, value }],   // already had that value - nothing to save
  refused:   [{ row, column, value, reason }] }
```

`row` / `column` in `staged` and `unchanged` are the readable names (matrix path and column label; flat row id and header). Always report `refused` to the user — a refused edit is not written. The reasons:

| `reason` | Means |
|---|---|
| `cell is not editable` | Locked for this user: a lockAndRelease aggregate, a subtotal, a calculated or locked row, a `customEditable: { editable: false }` cell |
| `edit type "…" / format "…" is left to the user` | Editable by hand but not by `stage()` — see section 9 |
| `table is not in edit mode and this user has no Edit button` | `lockedEdit` table the user cannot switch into edit mode |
| `row not found` / `column not found` | Reference matches nothing |
| `row reference matches more than one row` / `column reference …` | Non-unique label or field match — use the id, path or keys |
| `not a number: …`, `not a date - use yyyy-mm-dd`, `not a month - use yyyy-mm`, `not a colour - use #rrggbb`, `not valid JSON`, `not a bool value` | Value does not fit the cell |
| `not an option of "<dimId>" - see tablixPreview.options` / `not options of "<dimId>": …` | Unknown dropdown / multi-select value |
| `contains < or > - markup is not allowed …` | See section 5 |
| `rejected by validation: <message>` | The grid's own validation turned it down, with its message — required field (`Väärtus on kohustuslik`), a uniqueness rule (`Väärtus peab olema unikaalne`), a validation URL |

---

## 7. What the table does while changes are staged

- **The table defers every change, the user's own typing included.** A matrix normally saves on each edit (`saveOnChange`); while changes are staged it waits for Save, so the user can review their own additions alongside yours. The Save button appears for that time.
- **Saving, Reset, Close edit, `discard()` or a reload** end it and return the table to its own behaviour.
- **Staged changes are dropped, without a warning, when the table reloads** — a different department / employee picked in a selector, a filter change, a page reload. That is the grid's normal handling of unsaved changes. Stage after the user has settled on the selection, and tell them not to switch before saving.
- `data()` shows the staged values (values include unsaved changes) and lists them in `pending`: `{ rowId: [fields] }`.

---

## 8. Matrices with calculated totals — the numbers change on Save

A matrix's subtotals, parent rows and grand-total row are **not recalculated while changes are staged**. The preview shows only the cells you staged; everything computed from them keeps its old value until the user saves, and then the matrix redraws from the server's response. Tell the user this when you stage into such a matrix, so a total that "did not move" is not read as a failed edit.

- **Leaf cells, grand-total row.** Tab *Lame*: staging `01.26 › Qty` 25 → 30 and `01.26 › Amount` → 100 on one row left the grand-total row at `890` / `1 748` while staged; after Save it read `895` / `1 848`.
- **Editing a subtotal distributes it.** On a matrix with `allowTotalChange` (tab *Hierarhiline*, and `lockAndRelease` matrices generally), parent rows are editable too — they appear in `editable`. Staging the item subtotal `<customer> / <item>`, `01.26 › Amount` 88 → 110:
  - while staged: only that cell showed 110; its two children (22 and 66), the customer total above (`394`) and the grand total (`1 848`) were unchanged;
  - on Save one record went out with the subtotal's coordinates only (`cust: <customer>, item: <item>` — no business line) and `quantity: 110`;
  - the server spread it over the children in proportion (`22 → 28`, `66 → 82`), and the customer total (`394 → 416`) and grand total (`1 848 → 1 870`) followed.

  So a staged subtotal is a target the backend allocates, not a value that stays where you put it — children get rounded shares. When the user wants specific child values, stage the children, not the parent.
- **Formula measures and derived rows follow on Save too.** On a salary budget the Palgakulud / Tööjõumaksud rows (calculated from the Sisestusala input rows) came back filled only after Save.
- **Locked aggregate levels.** On a `lockAndRelease` budget, nothing is editable at an aggregate level of the selector (a division or department — 0 editable cells); pick a leaf (an employee) first. `data()` then shows the `editable` lists.

Flat tables have a related effect: a row with a `rowFormula` (e.g. `Blackberry: "Coconut*Cranberry"`) is recalculated in the page whenever a row it names changes, staged or typed. Those recalculated values are not pending and are not saved; a reload shows the stored ones. Its dropdown columns get multiplied too (`undef: 9259250625`) — an existing grid quirk, not a staging error.

---

## 9. Not covered — left to the user

- **Row-header dimension cells**: a stepped-layout matrix's dimension columns (`dimensionIds.cust`, …) and a hierarchical matrix's caption cell (`rowValueField`). Editing those moves a row to another customer/item/account, not a value.
- **Matrix dimension cells** (a matrix cell holding a dimension key, `editType: 'dropdown'`).
- Flat edit types other than input, textarea, dropdown, input-dropdown, toggle, monthpicker and btnSelect — e.g. `text` (the emoji picker column), `svg`, `customString`, forms in cells; and a column whose edit type is set by a cell condition to one of those.
- **Adding or deleting rows** — `stage()` only changes existing cells. To add rows use `stageRows()` or `insertRows()`; to delete them `await removeRows()`, which leaves a server delete to the user's confirmation unless they asked for `{ confirm: true }` (`references/forms.md` § 7). Toolbar buttons such as copy-row are in § 8 there (`toolbar()`, `action()`).
- **Dropdown options filtered by other columns** — `stage()` checks a value against the cell's whole dimension, as paste does, not against a narrower dependent list.

---

## 10. Examples

**Fill every editable month of one matrix row:**

```js
const p = window.tablixPreview;
const d = p.data('finTablix');
const row = d.rows.find(r => r.path?.join(' › ') === 'Sisestusala › Muud kulud › Autokulud');
const edits = (row.editable || []).map(label => ({ row: row.id, column: label, value: 180 }));
const res = await p.stage('finTablix', edits);
JSON.stringify({ staged: res.staged.length, refused: res.refused.length });
```

**A per-month plan across several rows** — compute the values, then stage once:

```js
const days = [21, 19, 22, 21, 20, 20, 23, 20, 22, 22, 21, 21];   // working days per month
const plan = { 'Põhipalk': () => 2800, 'Tööpäevi kuus': i => days[i % 12] };
const d = window.tablixPreview.data('finTablix');
const edits = [];
d.rows.filter(r => r.editable && plan[r.label]).forEach(r =>
  d.columns.forEach((c, i) => { if (r.editable.includes(c.label)) edits.push({ row: r.id, column: c.label, value: plan[r.label](i) }); }));
```

(`Põhipalk` is not unique — using `r.id` from the filtered rows sidesteps that.)

**Flat table, several types at once:**

```js
const p = window.tablixPreview;
const res = await p.stage('SisestusTest', [
  { row: { text: 'Banana' }, column: 'Staatus', value: 'Queued' },          // dropdown by label
  { row: { text: 'Banana' }, column: 'Kaal', value: 18.5 },
  { row: { text: 'Banana' }, column: 'Kpv', value: '2026-10-05' },
  { row: { text: 'Cherry' }, column: 'Protsent', value: '100%' },
  { row: { text: 'Coconut' }, column: 'Kategooria', value: ['puuvili', 'lehtköögivili'] },
  { row: { text: 'Coconut' }, column: 'Json', value: { name: 'Coconut', color: '#8b5a2b' } },
]);
```

---

## 11. Troubleshooting

| Symptom | Cause |
|---|---|
| `tablixPreview.stage is not a function` | Build predates it. |
| Every edit `cell is not editable` on a matrix | An aggregate level is selected (lockAndRelease), or the user has no edit rights. Check `data().rows.some(r => r.editable)`. |
| Staged, but a total did not change | Expected — totals and formulas recalculate on Save (section 8). |
| After Save a subtotal's children show different numbers than expected | The subtotal was distributed by the server (section 8). Stage the children instead. |
| Staged changes vanished | The table reloaded (selection, filter or page change) before Save (section 7). |
| `unchanged` where you expected a change | The cell already had that value — including a value that converts to the same thing (`'38%'` on a cell holding `0.38`). |

