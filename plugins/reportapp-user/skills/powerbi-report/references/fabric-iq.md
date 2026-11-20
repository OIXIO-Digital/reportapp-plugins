# Querying the semantic model — Fabric IQ preferred

`reportPreview` (`references/report-preview.md`) knows the **view**: which report, page, visuals, filters and slicers, what the user clicked, the numbers each visual shows. It does not query the semantic model. If the visuals fully answer the question, use them; a model query is not mandatory. For questions the visuals don't answer — another breakdown, a longer period, the rows behind a total, or actual year-to-date performance separately from a full-year forecast — use a permitted semantic-model connector, preferring Microsoft's **Fabric IQ MCP server**.

The two are complementary: Fabric IQ knows the **model** but not the user's view; `reportPreview` supplies the view so the query matches what the user is looking at.

---

## 1. Is it available?

### Selection and fallback policy

1. **Fabric IQ first for business-data questions.** Microsoft's [Power BI MCP overview](https://learn.microsoft.com/en-us/power-bi/developer/mcp/mcp-servers-overview) directs consumption scenarios to Fabric IQ. It explicitly says not to use the Power BI Authoring MCP server for end-user business questions, even though that server can execute DAX for model validation. This preference concerns model queries; it does not require querying the model when the report visuals already suffice.
2. **Approved authenticated alternatives are allowed when needed.** If Fabric IQ is unavailable or cannot support a required query, use another connected Power BI semantic-model connector only when it is approved for the user's organization and data handling, authenticates independently as the user, respects their permissions and model security, and supports read-only business-data queries. Microsoft's earlier Power BI Consumption MCP server is one possible existing integration. Do not treat an arbitrary tool as approved merely because it is installed or can run DAX; ask if approval is unclear. Do not use an authoring connector for this workflow.
3. **Do not bypass a refusal.** Explain authentication, permission, security or tenant-policy failures; do not switch connectors to evade them. An unsupported capability may justify an approved alternative, but never broader access.
4. **No suitable connector:** answer only what `visualData()` supports, state what remains unanswered and what a model query would add. Setting up a connector is the user's (or their admin's) choice — tell them it exists, don't push it.

For Fabric IQ, look for tools named `DiscoverArtifacts`, `ResolveFabricItem`, `GetReportMetadata`, `GetSemanticModelSchema`, `ValueSearch`, `ExecuteQuery` (endpoint `https://fabriciq.svc.cloud.microsoft/v1/mcp/FabricIQ`). Inspect the available tool definitions at runtime rather than assuming a schema.

The steps below use Fabric IQ tool names. With an approved alternative, use its documented discovery, schema and read-only query tools; the same filter, measure, reconciliation and citation requirements apply. State which connector supplied model-query results.

**Never** extract or reuse the page's Power BI access token, ReportApp credentials, `Authorization` header or `sessionId`; call Power BI or ReportApp REST APIs directly; or use page `fetch`/XHR or request interceptors. A connector's own authenticated implementation is not permission to make direct API calls outside it.

What Fabric IQ needs and gives (Microsoft Learn, *Get started with the Fabric IQ MCP server*):

- Read permission on the semantic model is enough — no Build permission, no workspace role. Row-level and object-level security apply, so the user gets their own numbers.
- Runs as the signed-in user (delegated sign-in in their AI tool). Not available in Power BI-only regions or sovereign clouds.
- `ExecuteQuery` returns 250 rows unless `maxRows` is set; aggregate in DAX instead of fetching rows.
- Results are as fresh as the model's last refresh.

---

## 2. From the page to the model

1. `await reportPreview.info()` → `powerBiUrl`.
2. `ResolveFabricItem` with that URL → the identifiers the other tools need. (Don't use a Power BI *share* link.)
   - `info().datasetId` (when not `null`) already names the semantic model in `info().workspaceId`. Pass it where a tool takes a model id and you can skip discovery. Still resolve the report URL when a tool needs the report's own identifiers.
   - With `datasetIdFrom: 'refresh setup'`, reconcile (§ 4) before trusting it: it is an admin-entered setting.
3. `GetReportMetadata` for the report's pages and visuals as the model sees them; `GetSemanticModelSchema` for tables, columns and measures. **Use the report's own measures** (`[Actual]`, `[Budget]`, …) — they carry the business logic; don't rebuild them from columns.
4. `ValueSearch` when the user names a value ("the Tallinn office", "PL1") to find how it is stored and in which column.
5. Build the DAX (§3) and run `ExecuteQuery`.
6. Reconcile (§4), then answer.

---

## 3. Writing the query with the user's filters

One call gives every filter in effect for the visual, as exact DAX, so nothing is copied by hand:

```js
const c = await reportPreview.daxContext('<visual name or label>', { selection: true /* only for "this point" questions */ });
JSON.stringify({ args: c.calculateTableArgs, untranslated: c.untranslated, fields: c.fields, complete: c.complete });
```

Paste `calculateTableArgs` as the filter arguments of `CALCULATETABLE`, around the measures you query (names from `GetSemanticModelSchema`):

```dax
EVALUATE
CALCULATETABLE (
    ADDCOLUMNS (
        VALUES ( 'Date'[Quarter] ),
        "Actual", [Actual],
        "Budget", [Budget]
    ),
    KEEPFILTERS('Financial Statement'[FinRepTypeName] IN { "P&L", "KPI" }),
    KEEPFILTERS(('Date'[Date] >= DATE(2024, 1, 1) && 'Date'[Date] <= DATE(2026, 10, 15))),
    KEEPFILTERS('Company'[Company] IN { "<company>" })
)
ORDER BY 'Date'[Quarter]
```

Rules:

- **Without a visual** (`daxContext()`), you get the report, page and slicer filters: what every visual on the page shares. **With a visual**, its own filters are added (e.g. a card limited to `Level1 = "Müügitulu"`). Use the visual form when you reproduce that visual's number.
- **`untranslated`** lists filters with no exact DAX form: relative dates, measure filters (evaluated at the visual's grain), TopN, tuple and hierarchy filters, a multi-column multi-point selection. **Do not approximate them.** `complete: false` means the query does not reproduce the screen. Say which filters the answer leaves out, or ask the user.
- **`fields`** are field-parameter choices, not row filters. `kind: 'grouping'` names the column the visuals group by (`'Date'[Quarter]`): group your query by it. `kind: 'measures'` names the measures the visuals show: query those.
- **Slicer interactions are not visible.** A report author can switch a slicer off for one visual, and Power BI doesn't expose that, so `daxContext` still lists that slicer. That is why § 4 is not optional.
- **The clicked point** comes in with `{ selection: true }`, as exact predicates (`ISBLANK(...)` for a blank member). Use it for "why is this bar low" questions, not for general ones.

### Field-parameter references

`setSlicer` on a field-parameter slicer needs references such as `'Date'[Quarter]`, but the slicer shows labels (Kuu, Kvartal, Aasta). The parameter is a table in the model (the filter's table name — e.g. `Date set`): query it with `EVALUATE '<table>'` and read the label and fields columns. If that doesn't give the reference, ask the user to click the option instead.

---

## 4. Reconcile before you answer

A query can drift from the screen (a missed filter, a visual-level filter, a measure that depends on the visual's grouping). Before quoting a number:

1. Reproduce one number the user can see — a card or a visible bar — with the same filters.
2. Compare it to `visualData()` for that visual.
3. If they differ, find the filter you missed (`filters({ visuals: true })`) or say plainly which number you trust and why.

Say which filters your answer uses ("for <company>, Jan 2024 – Oct 2026, P&L and KPI rows").

---

## 5. Citing the source

Fabric IQ can name the model, tables and measures; the page adds the report-level reference:

- the **report and page** (`info().name`, `activePage.displayName`) and the **visual label** (`visuals()` `label`, never a hidden title) the number comes from;
- the **filters** in effect, in words;
- a **link** (`reportPreview.link()`) that reopens the view — or `link({ slicers })` for the state your answer describes. Mention what it `omitted` and that opening it saves that view for the recipient.
