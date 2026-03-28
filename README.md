# b1-report-templates

A library of pre-built SAP Business One report templates. An AI agent (or any program) picks a template, injects live Service Layer data, and hands the user a complete self-contained HTML file — with charts, tables, and deeplinks into the SAP Web Client.

Hosted via GitHub Pages at:
```
https://<your-username>.github.io/b1-report-templates/manifest.json
```

The consuming program hardcodes only that one URL. Template updates and new reports are available immediately on push — no extension or client update needed.

---

## How it works

```
1. Program fetches manifest.json         → discovers available templates
2. Matches user intent to a template     → by id, tags, or description
3. Runs the SL queries in the manifest   → gets live JSON data
4. Fetches the template HTML             → pre-designed, pre-built
5. Calls window.B1Report.load(data)      → template renders itself
6. Delivers the HTML file to the user    → self-contained, no runtime deps
```

The user receives a finished HTML file. Charts are rendered, tables are populated, every record links back to the correct screen in the SAP Web Client. They can save it, share it, or open it offline.

---

## Repository structure

```
b1-report-templates/
├── manifest.json               ← registry of all templates + their SL queries
├── _template-starter.html      ← copy this to build a new template
├── sales-performance.html
├── ar-aging.html
├── inventory-levels.html
└── purchase-spend.html
```

---

## Adding a new template

### 1. Copy the starter

```bash
cp _template-starter.html my-new-report.html
```

### 2. Build the report

Design the full report in the HTML file — layout, charts, tables, colours. Everything is pre-built; the only dynamic part is the data the agent injects.

Every template must expose `window.B1Report` with this contract:

```js
window.B1Report = {
  meta: {
    id:          "my-new-report",
    name:        "Human readable name",
    description: "Shown to the AI for intent matching — be specific about what this covers",
    tags:        ["keyword1", "keyword2"],
    queries:     ["primaryData", "secondaryData"]  // must match query keys in manifest
  },

  load(data) {
    // data.primaryData   = SL value array
    // data.secondaryData = SL value array or SQL result
    // data.company       = string e.g. "CRONUS_DK"
    // data.webClientBase = base URL e.g. "https://sap-server:8080"
    //
    // Render charts, populate tables, build deeplinks here.
    // This may be called more than once (e.g. refresh) — destroy
    // and recreate Chart.js instances rather than appending.
  }
};
```

### 3. Add demo mode

So the template opens correctly in a browser without a live SL connection — useful for design and testing:

```js
(function demoIfStandalone() {
  if (window.__B1_AGENT_CONTROLLED__) return; // skip when agent is driving
  window.B1Report.load({
    primaryData: [ /* realistic mock rows */ ],
    company: "DEMO_DB (demo)",
    webClientBase: "https://your-sap-server:8080"
  });
})();
```

### 4. Build SAP Web Client deeplinks

Use `data.webClientBase` so links point to the right server:

```js
// Invoice deeplink
const url = `${data.webClientBase}/webapp/index.html#/OINV/${row.DocEntry}`;
link.href = url;

// Customer deeplink
const url = `${data.webClientBase}/webapp/index.html#/OCRD/${row.CardCode}`;
```

Common SAP Web Client URL patterns:

| Object | URL pattern |
|---|---|
| Invoice (OINV) | `/webapp/index.html#/OINV/{DocEntry}` |
| Customer (OCRD) | `/webapp/index.html#/OCRD/{CardCode}` |
| Purchase order (OPOR) | `/webapp/index.html#/OPOR/{DocEntry}` |
| Item (OITM) | `/webapp/index.html#/OITM/{ItemCode}` |
| Delivery (ODLN) | `/webapp/index.html#/ODLN/{DocEntry}` |

### 5. Register in manifest.json

```json
{
  "id": "my-new-report",
  "name": "My new report",
  "description": "What this report shows — the AI reads this to pick the right template",
  "url": "https://<your-username>.github.io/b1-report-templates/my-new-report.html",
  "tags": ["keyword1", "keyword2"],
  "queries": [
    {
      "key": "primaryData",
      "endpoint": "OINV",
      "select": "DocEntry,DocNum,CardCode,CardName,DocDate,DocTotal,PaidToDate,DocCur,DocStatus",
      "filter": "DocDate ge '{date_6m_ago}' and Canceled eq 'N'",
      "orderby": "DocDate desc",
      "top": 200
    },
    {
      "key": "secondaryData",
      "sql": "SELECT MONTH(T0.\"DocDate\") AS Mo, SUM(T0.\"DocTotal\") AS Revenue FROM OINV T0 WHERE T0.\"DocDate\" >= DATEADD(MONTH,-6,GETDATE()) GROUP BY MONTH(T0.\"DocDate\")"
    }
  ]
}
```

### 6. Push

```bash
git add my-new-report.html manifest.json
git commit -m "add my new report"
git push
```

Live in ~30 seconds. No client update needed.

---

## manifest.json field reference

```jsonc
{
  "version": 1,
  "updated": "2025-03-27",
  "templates": [
    {
      "id": "string",           // unique slug — matches meta.id in the HTML file
      "name": "string",         // short display name
      "description": "string",  // used by LLM for intent matching — be descriptive
      "url": "string",          // full GitHub Pages URL to the .html file
      "tags": ["string"],       // keywords to help intent matching
      "queries": [
        // OData entity query
        {
          "key": "string",      // key in data object passed to load()
          "endpoint": "OINV",   // SL entity
          "select": "...",      // $select
          "filter": "...",      // $filter — supports placeholders (see below)
          "orderby": "...",     // $orderby
          "top": 100            // $top
        },
        // SQL query (for aggregations and joins)
        {
          "key": "string",
          "sql": "SELECT ..."
        }
      ]
    }
  ]
}
```

### Filter placeholders

| Placeholder | Value |
|---|---|
| `{date_6m_ago}` | ISO date 6 months before today |
| `{date_3m_ago}` | ISO date 3 months before today |
| `{date_1m_ago}` | ISO date 1 month before today |
| `{date_1y_ago}` | ISO date 1 year before today |
| `{today}` | Today's ISO date |

---

## What the agent passes to load()

```js
{
  // One key per query defined in the manifest, populated with SL response .value arrays
  invoices: [ { DocEntry, DocNum, CardName, DocTotal, ... }, ... ],
  monthly:  [ { Mo, Yr, Revenue }, ... ],

  // Always included
  company:       "CRONUS_DK",            // current company DB name
  webClientBase: "https://sap:8080"      // base URL for building deeplinks
}
```

---

## Current templates

| ID | Name | Covers |
|---|---|---|
| `sales-performance` | Sales performance | Revenue, top customers, invoice status (OINV) |
| `ar-aging` | AR aging | Overdue buckets, collections priority (OINV open) |
| `inventory-levels` | Inventory levels | Stock on hand, reorder alerts (OITW, OITM) |
| `purchase-spend` | Purchase & vendor spend | PO volume, top vendors (OPOR) |

---

## Design conventions

All templates share the same visual language so reports look consistent regardless of which one is used:

- Dark theme — `#0f1117` background
- Fonts — DM Sans (UI) + DM Mono (numbers, codes, labels)
- Charts — Chart.js 4, consistent tooltip and axis styling
- Colour coding — blue (primary metric), green (positive/paid), amber (warning/open), red (overdue/danger)
- Every record in a table links to the corresponding SAP Web Client screen
- Templates are fully self-contained single-file HTML — no build step, no external dependencies at runtime
