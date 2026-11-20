# Forms, thread comments and new rows — `tablixPreview.openForm()`

How to use a dashboard's **forms** on the user's behalf — see which ones the page offers, open one the way its button does, fill it in, and leave it for the user to check and submit. Also how to post a message to a discussion thread and how to insert several rows. Companion to `references/table-editing.md` (changing values in existing cells) and `references/table-data.md` (reading).

**Never fill a form through the DOM for this** (clicking the sidebar, typing into inputs). The calls below open the form through the same code as its button and put each value in through the control's own change handler, so dependent fields, show/hide conditions and change tracking react exactly as to typing — and every value is checked, with the refusals reported.

---

## 1. The rule: fill, the user submits

`openForm()` opens a form and fills it. **Nothing is sent.** The user checks the values and presses the form's own button (Lisa / Add / Save), or cancels.

Submit yourself — `submitForm()`, `insertRows()`, `comment({ submit: true })` — **only when the user has said so in this conversation** ("just add them", "post it", "no need to show me"). Those go through the form's own submit path, so its required fields, uniqueness rules and validation URL still apply; the user just doesn't see the form first. When in doubt, fill and stop.

---

## 2. Availability

- **A build that has it.** Probe `typeof window.tablixPreview?.openForm === 'function'`. Missing means the deployed build predates it — say so; do not fall back to DOM typing.
- **Not admin-gated.** A form opens for whoever could open it by hand; a read-only user gets the same refusals the buttons give.
- **On a `/grid/IndexPage?configId=<id>` dashboard.**

---

## 3. Overview — which forms the page has

```js
const p = window.tablixPreview;
JSON.stringify(p.forms().map(f => ({ id: f.id, name: f.name, from: f.opensFrom.map(w => w.kind), ready: f.ready, blocked: f.blocked })));
```

Each entry:

| Key | Means |
|---|---|
| `id`, `name` | The form. `id` is what every other call takes. |
| `opensFrom` | Every way the page opens it — see the table below. A form can have several. |
| `placement` | `'modal'` (the shared pop-up) or `'container:<id>'` (a tab in a card). |
| `open` | Drawn and visible right now. |
| `submitsTo` | `{ url, method }` it posts to. |
| `submitsOnOpen` | Its button submits straight away (`instantSubmit`, or a sidebar entry without fields) — nothing to fill. |
| `fieldsLoadedOnOpen` | Its fields come from the server when it opens (`metadataUrl`) — call `form(id)` again after opening for the exact set. |
| `required` | Fields that must be filled. |
| `needs` / `sends` | What opening depends on (a selected row, selected matrix rows), and what goes along with its fields (the applied filters, the selected matrix cells, a table's selection). |
| `ready` / `blocked` | Whether it can be opened now, and why not. |

`opensFrom[].kind`:

| `kind` | Opened by | Notes |
|---|---|---|
| `sidebar` | an entry in a sidebar group (`group` names it) | |
| `toolbar` | a button on a table's or form's toolbar | `contextMenu: true` when it is also in the right-click menu |
| `addRow` | the `+` of a table (`table`) | `hierarchical: true` → rows can be added under a parent (`parent`, section 4) |
| `edit` | selecting a row in `table` | edits that row — pass `row` (section 4) |
| `container` | its tab in a card | |
| `rowForm` | — | drawn **inside** the rows of `table` (a card-table's `form.view` / `form.edit`); not opened on its own |
| `import` | the table's Excel import button | needs a file from the user — not opened |

### One form's fields

```js
JSON.stringify(p.form(2040).fields);
```

Each field: `field` (the `valueField` — what `openForm` takes), `label`, `editType`, `formatType`, `required`, `fillable` (and `why` when not), `dimId`, `multiple`, `placeholder`, `defaultValue`, and `shownWhen: { field, values, effect }` when a condition group shows (or unlocks) it only for some value of another field. When the form is open, also `value`, `visible` and `error`.

`form(id)` itself also says, while an edit form is open, **which row it holds**: `editing: { table, row }` (the row's id). An add form, or an edit form holding a new object with no id yet, has no `editing`. Read it before filling an edit form you did not just open yourself — a master-detail form shows whatever row is selected, which may not be the one you mean (section 4, `{ row }` guard).

`p.formOptions(formId, field)` returns `{ dimId, multiple, freeText, options: [{ value, label }] }` for a dropdown field, `null` for one without a dimension.

---

## 4. Opening and filling

```js
const r = await p.openForm(2040, { budgetId: 'Kuu eelarve', fromPeriod: '2026-04', periodCount: 3, department: ['Administration'] });
JSON.stringify({ opened: r.opened, reason: r.reason, filled: r.filled.length, refused: r.refused, missing: r.missingRequired });
```

`openForm(formId, values, opts)` — `values` is `{ field: value }`, field by `valueField` or by a unique label. `opts`:

| Option | For |
|---|---|
| `via` | Which way to open it when `opensFrom` lists several (`'addRow'`, `'sidebar'`, …). Default: the first. |
| `row` | **Edit forms** — the row to edit: its id, or `{ field: value }` matching exactly one row (`{ code: 'DEV' }`). A row under collapsed parents has them expanded first (builds from 2026-10-09; older ones answer `row is not drawn … expand it first`). It is selected in its table first, as a click does, and a form behind an Edit button is switched into edit mode. |
| `parent` | **Add-row forms of a hierarchical table** — the row to add under. Without it the new row goes on the top level. |
| `unappliedFilters` | `'apply'` or `'ignore'`. When the user changed filters without pressing Set filter, the form's button asks what to do. A call cannot answer that dialog, so `openForm` returns the question instead — ask the user, then pass their answer: `'apply'` applies the filters first, `'ignore'` uses the ones already applied. |

The result:

```js
{ opened: true, formId,
  filled:  [{ field, value }],            // stored value, read back from the form
  refused: [{ field, value, reason }],    // not written
  missingRequired: [field] }              // required, visible and still empty
// or
{ opened: false, formId, reason }
```

Always report `refused` and `missingRequired` to the user. `p.fillForm(formId, values, opts)` fills more fields of a form that is already open; `p.closeForm(formId)` closes it without submitting (a tab form stays on its tab with the edits thrown away, and one behind an Edit button leaves edit mode).

**Name the row on every call to an edit form.** `fillForm(formId, values, { row })` and `submitForm(formId, { row })` check that the open edit form holds that row (id or `{ field: value }`, as for `openForm`) and **refuse without touching anything** when it holds another — `reason: 'form "…" holds row 11 of "scheduler", not 23 - open that row with openForm(..., { row })'`. Without `{ row }` they act on whichever row the form shows. This matters most in master-detail layouts (a form that follows a table's selection), where an earlier `openForm` that did not reach its row, a user's click, or a refresh can leave a different record in the form. Passing `{ row }` to a form that edits no row (an add form) is refused too: `form "…" does not edit a row - leave out { row }`.

### Values

Give values the way a user would type them — the rules are the same as `stage()`'s (`references/table-editing.md` § 5). Where a form stores differently from a grid cell:

| Field | Give | Stored |
|---|---|---|
| dropdown | the option's `value` (id) or `label` from `formOptions()` | the id |
| multi-select (`dropdownMulti`) | an array, or a `','`-joined string, of ids or labels | `'275,139'` |
| multi-select (`multiple`, other format) | the same | an array of ids |
| toggle | `true` / `false` (also `1`/`0`, `'yes'`/`'no'`) | `true` / `false` |
| month picker | `'2026-04'`, `'04.2026'` or `'20260401'` | `'20260401'` |
| number / int | `12`, `'1 234,5'` — a plain number | number |
| percentage | **as shown**: `38` or `'38%'` | `0.38` |
| date / date-time | `'2026-04-15'` / an ISO string | as given |
| rich text | HTML — sanitised like the editor's output | sanitised HTML |
| cron (`editType: 'cron'`) | a 5-field expression, `'15 7 * * 1-5'` | as given — written through the cron builder, which redraws its fields, the description ("Kell 07:15, esmaspäevast reedeni") and the next runs. Not checked when filled: an invalid expression is refused by the form's own submit (`fields: ['schedule']`) |
| text / textarea | plain text — `<` and `>` refused only on a line with `allowAngleBrackets: false` (every text line on builds before 2026-10-20) | as given |

Not filled — left to the user, reported in `refused`: Excel uploads, button selects, display-only lines, fields filled by the system on submit (`systemGenerated`), and fields that are not editable or are locked. A table field whose grid is a table of its own on the page (`tableConfigRefId`) is refused with a pointer to `stage` / `stageRows` / `removeRows` on that table.

### Refusal reasons

| `reason` | Means |
|---|---|
| `no such field - see tablixPreview.form` | Unknown `valueField`, or a label that is not unique |
| `a file upload - left to the user`, `a table of its own on the page - …`, `display only`, `not editable`, `filled in by the system on submit (…)` | The field cannot be filled (above) |
| `a table field takes an array of rows … or { remove, update, add }` | A table field given something else |
| `row not found in "<table>"` | A table-field row reference matched nothing (or more than one row) |
| `field is locked` | Locked by a condition group or the form's locked state |
| `not a number`, `not a month - use yyyy-mm`, `not a date - use yyyy-mm-dd`, `not a toggle value - …`, `not valid JSON` | The value does not fit |
| `contains < or > - markup is not allowed in a text cell` (JSON: `… here`) | The field or table-field column has `allowAngleBrackets: false` — or the build predates 2026-10-20, when every text, textarea, input-dropdown and JSON cell refused `<` and `>`, the FRD formula grid included (`references/table-editing.md` § 5). Hand the text to the user. |
| `not an option of "<dimId>" - …` / `not options of "<dimId>": …` | Unknown dropdown value — see `formOptions()` |
| `the control did not take the value (stored: …)` | Written but read back differently — tell the user, do not retry blindly |

**A refusal does not hold back `submitForm()`.** `fillForm` and `submitForm` are separate calls: `submitForm` runs the form's own submit on whatever the form holds and returns `submitted: true` — a field that `fillForm` (or `openForm`) refused just keeps its old value, and nothing in the submit result says so. Only the internal open-and-submit path behind `insertRows` and `comment({ submit: true })` stops on `refused` / `missingRequired`. So when the user has asked for a submit: check the fill's `refused` and `missingRequired` first and stop if either is non-empty, and after the submit read the record back (`form(id).fields` → `value`, or the table's `data()`) before reporting that it changed.

`opened: false` reasons: the unapplied-filters question (above); `its selection requirements are not met` (see `needs`); `select exactly one row of "…" to edit - pass { row }`; `row …` / `parent not found in "…"` (the reference matched nothing or more than one row); `this form submits as soon as it is opened …`; `an Excel import needs a file from the user …`; `this form is drawn inside table rows …`.

---

## 5. Selecting cells or a row — `select()`

Forms that send a selection (`sends` lists it) send whatever is selected when they are submitted. Select it first:

```js
await p.select('finTablix', [{ row: ['Sisestusala', 'Lisatasu', 'Boonus summas'], column: '03.26 › Eelarve' }]);   // matrix cells
await p.select('FinBudgetTablix', { code: 'DIV2' });                                                              // one flat-table row
```

Matrix cells are addressed like `stage()` (`references/table-editing.md` § 4); a flat table takes one row reference. The selection is made the way a click makes it, so tables that follow the selection (`updateOnSelect`) reload as they would. Rows must be drawn — expand a collapsed parent first. Returns `{ selected, refused }` (matrix) or `{ selected, row, reason }` (flat).

---

## 6. Comments in a thread — `comment()`

A discussion thread is a hierarchical table of messages with an add-row form, like the "Sõnumid" panel (table `thread`). Its form sends the selected matrix cells, which is how a message belongs to the cells it is about:

```js
// after staging changes, explain them on those cells - left open for the user to send
await p.comment({
  subject: 'Boonused märts',
  message: '<p>Boonus summas 03.26 muudetud 4 299 → 3 800, et püsida eelarves.</p>',
  cells: { table: 'finTablix', at: [{ row: ['Sisestusala', 'Lisatasu', 'Boonus summas'], column: '03.26 › Eelarve' }] }
});

await p.comment({ replyTo: 40, message: 'Kontrollitud.' });                 // a reply
await p.comment({ replyTo: { subject: '2026 Budget process' }, message: '…' });
```

| Option | |
|---|---|
| `message` | The text (rich text — HTML allowed, sanitised) |
| `subject` | A new thread's subject. Replies have none — the subject field is hidden for them |
| `replyTo` | The message to answer: its id, or `{ field: value }` matching one message |
| `cells` | `{ table, at }` — cells to select first (section 5) |
| `values` | Any other fields of the thread form (`{ approvalStatus: 'Info' }`) |
| `table` | The thread table, when it is not called `thread` |
| `submit` | `true` posts it — only when the user said to (section 1). Default: left open |
| `unappliedFilters` | As for `openForm` |

**The thread follows the page's selection.** On a budget dashboard the thread shows the messages of the department picked on the left; with another one selected, `replyTo: 40` comes back `parent not found in "thread"`. Select the right row first.

---

## 7. Inserting several rows

Pick by whether the user wants to review them:

| Call | The user reviews first | Requests | Needs |
|---|---|---|---|
| `stageRows(tableId, rows)` | **yes** — rows appear in the grid, the user presses the toolbar Save | one, on Save | a table with `inlineNew` and a `saveUrl` — or a grid inside a form, saved with the form |
| `insertRows(tableId, rows, { batch: true })` | no | one | an add-row form with `saveUrl` and `saveAsArray` |
| `insertRows(tableId, rows)` | no | one per row | any add-row form |
| Excel import | — | — | the user's own file; point them to the table's import button |

`rows` is an array of `{ field: value }`, field by `valueField` or header/label, values as in section 4. All three take `{ parent }` for a hierarchical table.

**`stageRows()` — the review option.** Each row is created as the table's own "+ New" creates one, and its values go in through `stage()`, so they are checked, converted and outlined green like any staged cell; the table waits for Save. Returns `{ staged, rows: [{ index, added, id, staged, refused, missingRequired }] }`. A row where nothing could be filled is taken out again (`added: false`). `tablixPreview.discard(tableId)` removes the new rows, as well as any other unsaved change. A table without `inlineNew` refuses with a reason that points to `insertRows`.

**`insertRows(…, { batch: true })`** checks every row the way the add-row form checks one — values, required fields (only those shown for that row's values), the table's uniqueness rules — and sends all of them in one request, as a paste past the end of a table does. **Nothing is sent unless every row passes:** otherwise it returns `{ inserted: 0, problems: [{ index, row, refused, missingRequired, conflict }] }` for you to fix and retry. Fields a row does not give get the form's defaults.

**`insertRows(…)`** opens, fills and submits the add-row form once per row, stopping at the first row that is refused or fails — `{ inserted, stoppedAt, row, reason, refused, missingRequired }` — and leaves that row's form open. Slower (a few seconds a row); use it when the batch is not available.

Both `insertRows` forms submit — section 1 applies.

**Removing rows — `await removeRows(tableId, rows, opts)`** does what each row's trash icon does. It is **async** (builds from 2026-10-14) — await it. Rows are ids or `{ field: value }`, each matching one row.

| Table | Without `opts` | With `{ confirm: true }` |
|---|---|---|
| Unsaved new rows (any table) | removed at once | the same |
| A grid inside a form (above) | saved rows go too; the form sends what is left on its save | the same |
| A table that deletes on the server (`deleteUrl`) | the delete confirmation opens; result `{ removed, awaitingConfirmation: [ids], next }` — the user confirms it | **deleted on the server at once**, as the confirmation's Delete button does, no dialog; result `{ removed, deleted: [ids], refused }` |

`confirm: true` deletes without the user seeing the dialog — section 1 applies: only when the user said to delete those rows. If the server delete fails, nothing is reported as deleted and the rows come back in `refused` with the reason. Refused per row: `row not found in "<table>"`, `reference matches more than one row`, `row is not deletable`; for the whole call: a table without `deleteEnabled` (`rows of "…" cannot be deleted`) and a matrix (`removeRows is for flat tables`).

**After a Save, a new row may show `id: 0` until the table is refreshed.** Some table APIs answer a save with `[]` instead of the saved rows; the grid then falls back to the rows it sent, whose ids are all `0`, and they collapse into one. The rows are stored correctly — refresh the table (↻) to see them. The table's own "+ New" behaves the same way.

---

## 8. Toolbar buttons — `toolbar()` and `action()`

**What a table's card offers this user, and how to reach each button without clicking it:**

```js
JSON.stringify(p.toolbar('scheduler'));
// { table: 'scheduler', drawn: true, buttons: [
//   { action: 'refresh',     label: 'Uuenda andmed',        use: 'action' },
//   { action: 'add-row',     label: 'Lisa uus',             use: 'openForm (opensFrom addRow) / insertRows / stageRows' },
//   { action: 'clone-row',   label: 'Kopeeri rida',         use: 'action' },
//   { action: 'delete-rows', label: 'Kustuta valitud read', use: 'removeRows' },
//   { action: 'reset-bookmark', label: 'Taasta vaikesätted', disabled: true, use: 'screen only' }, … ] }
```

A button is listed when the page offers it to this user — also when a narrow card has moved it into the "…" overflow menu. Hidden buttons (not allowed, or edit-mode buttons outside edit mode) are not listed. `disabled: true` means it is drawn but greyed out right now (e.g. `clone-row` with no row selected). `use` says which call covers it:

| `use` | Means |
|---|---|
| `action` | Run it with `action()` (below) |
| a call name (`removeRows`, `openForm …`, `stage`) | That call already does it, with checks and a result — use it instead |
| `needs a file from the user` | Excel import — point the user to the button |
| `not callable yet - starts a move the user finishes` | `move-up`/`move-down`/`level-up`/`level-down` — leave to the user |
| `screen only` | Changes only what is on screen (search bar, setup, full screen, hide) |

**A table drawn inside a form has no toolbar of its own.** The card shows the form, and its Save / refresh are the form's. `toolbar()` then returns `buttons: []` with `belongsTo: { type: 'form', id }` and a reason pointing to `submitForm` (save) and `closeForm` (throw changes away). `drawn: false` means the table's card is not visible.

**Running a button — `await p.action(tableId, name, { row })`** runs it the way its click does. Only buttons whose `use` is `'action'`, and only while `toolbar()` lists them enabled for this user:

| `name` | Does | Writes to the server |
|---|---|---|
| `refresh` | Reloads the table's data | no |
| `clone-row` | Copies the selected row — pass `{ row }` to select it first | **yes** — the copy is created at once |
| `save-changes` | The toolbar Save: sends the table's unsaved changes | **yes** |
| `reset-table` | Throws the table's unsaved changes away | no |
| `undo` / `redo` | One step of the table's undo history | no |

`{ row }` (id or `{ field: value }`) selects that row first, as a click does. Returns `{ done: true, action, rows, added }` — `added` lists the ids of rows that appeared (the clone's new id). Refusals: `"<name>" is not on the toolbar of "<table>" for this user`, `"<name>" is not run through action - use: …`, `"<name>" is disabled right now` (for `clone-row`: select a row, or pass `{ row }`), and the `row …` reasons of `openForm`.

`clone-row` and `save-changes` write — section 1 applies: only when the user asked for it.

---

## 9. Troubleshooting

| Symptom | Cause |
|---|---|
| `openForm is not a function` | Build predates it. |
| `opened: false`, "filters were changed but not applied" | Ask the user whether to apply them, then pass `unappliedFilters` (section 4). |
| A form in `forms()` has an empty `opensFrom` | Nothing on the page opens it — an internal or orphaned form. |
| A required field stays in `missingRequired` after filling | It was refused (see `refused`), or it depends on another field's value — check `shownWhen`. |
| `parent not found` on a reply | The thread shows another selection's messages (section 6). |
| A value was refused as `the control did not take the value` | The form's own logic changed it (a field influence, a default). Read `form(id)` and tell the user. |
| New rows show `id: 0` after Save | Section 7 — refresh. |
| `fillForm` / `submitForm` refused: `form "…" holds row X …, not Y` | The edit form shows another row. `openForm(formId, {}, { row: Y })` first (section 4). |
| `removeRows` returned a Promise / `[object Promise]` | It is async since 2026-10-14 — `await` it. |
| `toolbar()` of a form's grid has no buttons | The card's toolbar is the form's — `belongsTo` (section 8). |
| `submitted: true`, but the record did not change | A field was refused when filling and the submit sent its old value (section 4, "A refusal does not hold back `submitForm()`"). Read the record back after every submit. |

