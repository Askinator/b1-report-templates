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
├── manifest.json                ← registry of all templates + their SL queries
├── _template-starter.html       ← copy this to build a new template
├── sales-performance.html
├── ar-aging.html
├── inventory-levels.html
├── purchase-spend.html
├── open-sales-orders.html
├── ap-payment-schedule.html
├── credit-limit-exposure.html
├── gross-margin.html
├── quotation-pipeline.html
├── returns-credit-notes.html
└── slow-moving-inventory.html
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
    // data.primaryData   = rows from the OData or SQL query
    // data.secondaryData = rows from the second query
    // data.company       = string e.g. "CRONUS_DK"
    // data.currency      = string e.g. "DKK"
    // data.webClientBase = base URL e.g. "https://sap-server:8080"
    // data.deeplinks     = deeplink definitions from manifest (see below)
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
    currency: "DKK",
    webClientBase: "",
    deeplinks: { invoice: { objName: "Invoices", key: "DocEntry" } }
  });
})();
```

### 4. Build SAP Web Client deeplinks

Deeplinks are built using `buildObjectUrl`, which is injected by the agent. Use the `sapLink` helper pattern to look up the right definition from `data.deeplinks`:

```js
const sapLink = (type, value) => {
  const def = deeplinks[type];
  if (!def || !value) return '';
  return B1Report.buildObjectUrl(def.objName, value);
};

// In your table rows:
const link = sapLink('invoice', row.DocEntry);
tr.onclick = () => { if (link) window.open(link, '_blank'); };
```

Deeplink types available by default (defined in `manifest.json` under `deeplinks`):

| Key | Object |
|---|---|
| `invoice` | Sales invoice (OINV) |
| `customer` | Business partner (OCRD) |
| `purchaseOrder` | Purchase order (OPOR) |
| `purchaseInvoice` | Purchase invoice (OPCH) |
| `item` | Item master (OITM) |
| `payment` | Incoming payment (ORCT) |
| `order` | Sales order (ORDR) |
| `quotation` | Quotation (OQUT) |
| `creditNote` | Credit note (ORIN) |

All deeplinks are resolved server-side — the template doesn't need to know or hardcode SAP Web Client URL patterns.

### 5. Register in manifest.json

```json
{
  "id": "my-new-report",
  "type": "standalone",
  "name": "My new report",
  "description": "What this report shows — the AI reads this to pick the right template",
  "tags": ["keyword1", "keyword2"],
  "queries": [
    {
      "key": "primaryData",
      "endpoint": "Invoices",
      "select": "DocEntry,DocNum,CardCode,CardName,DocDate,DocTotal,PaidToDate,DocCurrency,DocumentStatus",
      "filter": "DocDate ge '{date_6m_ago}' and Cancelled eq 'tNO'",
      "orderby": "DocDate desc",
      "top": 200
    },
    {
      "key": "summary",
      "sql": "SELECT T0.\"CardCode\", SUM(T0.\"DocTotal\") AS \"Total\" FROM OINV T0 WHERE T0.\"CANCELED\" = 'N' GROUP BY T0.\"CardCode\""
    }
  ],
  "aiHints": {
    "kpis": ["total revenue", "invoice count"],
    "charts": ["monthly revenue bar", "status donut"],
    "tables": ["all invoices sorted by date"],
    "deeplinks": ["invoice by DocEntry", "customer by CardCode"]
  }
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
  "version": 1.2,
  "updated": "2026-03-29",

  // Global deeplink type definitions — passed to every template as data.deeplinks
  "deeplinks": {
    "invoice": { "objName": "Invoices", "key": "DocEntry" }
    // key    = name used in sapLink() calls inside templates
    // objName = Service Layer entity name used by buildObjectUrl()
  },

  "reports": [
    {
      "id": "string",           // unique slug — matches meta.id in the HTML file
      "type": "standalone",     // currently always "standalone"
      "name": "string",         // short display name
      "description": "string",  // used by LLM for intent matching — be descriptive
      "tags": ["string"],       // keywords to help intent matching
      "queries": [
        // OData entity query
        {
          "key": "string",      // key in data object passed to load()
          "endpoint": "Invoices", // SL entity name (e.g. Invoices, BusinessPartners)
          "select": "...",      // $select
          "filter": "...",      // $filter — supports date placeholders (see below)
          "orderby": "...",     // $orderby
          "top": 100            // $top
        },
        // SQL query (for aggregations or joins not possible via OData)
        {
          "key": "string",
          "sql": "SELECT ..."   // see SQL restrictions below
        }
      ],
      "aiHints": {
        "kpis": ["string"],     // KPI descriptions for AI context
        "charts": ["string"],   // chart descriptions for AI context
        "tables": ["string"],   // table descriptions for AI context
        "warn": {},             // threshold values that trigger warnings
        "deeplinks": ["string"],// which deeplinks this report uses
        "compute": "string",    // note any values derived in JS, not from SL
        "notes": "string"       // any other relevant notes for the AI
      }
    }
  ]
}
```

### Date placeholders

Supported in both OData `filter` and SQL `sql` fields:

| Placeholder | Value |
|---|---|
| `{today}` | Today's ISO date |
| `{date_1m_ago}` | ISO date 1 month before today |
| `{date_3m_ago}` | ISO date 3 months before today |
| `{date_6m_ago}` | ISO date 6 months before today |
| `{date_1y_ago}` | ISO date 1 year before today |

### SQL query restrictions

The SAP Business One Service Layer SQL query endpoint has a restricted parser. Avoid:

- **Arithmetic inside aggregate functions** — `SUM(col1 * col2)` is rejected. Compute derived values in JavaScript instead, or use a wrapping subquery.
- **Arithmetic between aggregates in SELECT** — `SUM(col1) - SUM(col2)` is also rejected. Again, derive in JS.
- **`ORDER BY` after `GROUP BY`** — rejected by the parser. Sort results in JavaScript.

For columns that require arithmetic (e.g. cost = revenue − gross profit), return the raw aggregates from SQL and compute the derived value in `load()`.

---

## What the agent passes to `load()`

```js
{
  // One key per query defined in the manifest
  invoices:  [ { DocEntry, DocNum, CardName, DocTotal, ... }, ... ],  // OData rows
  summary:   [ { CardCode, Total }, ... ],                            // SQL rows

  // Always included
  company:       "CRONUS_DK",         // current company DB name
  currency:      "DKK",               // company default currency
  webClientBase: "https://sap:8080",  // base URL for building deeplinks

  // Deeplink definitions from manifest — use with sapLink() helper
  deeplinks: {
    invoice:  { objName: "Invoices",        key: "DocEntry" },
    customer: { objName: "BusinessPartners", key: "CardCode" },
    // ... all entries from manifest.json deeplinks
  }
}
```

---

## Current templates

| ID | Name | Data source | Covers |
|---|---|---|---|
| `sales-performance` | Sales performance | OData | Revenue trend, top customers, invoice status — rolling 6 months |
| `ar-aging` | AR aging | OData | Receivables split into aging buckets: current, 1–30, 31–60, 61–90, 90+ days |
| `inventory-levels` | Inventory levels | OData | Stock on hand by item, reorder alerts for items below minimum |
| `purchase-spend` | Purchase & vendor spend | OData | PO volume and vendor spend — rolling 6 months |
| `open-sales-orders` | Open sales orders | OData | Delivery backlog, days late, fulfillment priority |
| `ap-payment-schedule` | AP payment schedule | OData | Vendor invoices due in 7/14/30 days + overdue |
| `credit-limit-exposure` | Credit limit exposure | OData | Customers approaching or exceeding credit limit |
| `gross-margin` | Gross margin by customer | SQL | Customer profitability: revenue, cost, margin % — flags below-threshold customers |
| `quotation-pipeline` | Quotation pipeline | OData | Open quotes not yet converted, age, expiry tracking |
| `returns-credit-notes` | Returns & credit notes | OData | AR credit notes, return trends, repeat customers — rolling 6 months |
| `slow-moving-inventory` | Slow-moving inventory | SQL | Items with stock but no recent sales, tied-up capital |

---

## Design conventions

All templates share the same visual language so reports look consistent regardless of which one is used:

- Light theme — `#f0f2f7` background, `#ffffff` card surfaces
- Fonts — DM Sans (UI) + DM Mono (numbers, codes, labels)
- Charts — Chart.js 4, consistent tooltip and axis styling
- Colour coding — blue (primary metric), green (positive/healthy), amber (warning/open), red (overdue/danger)
- KPI cards have a 2px colour stripe at the top matching the metric's semantic colour
- Every record in a table links to the corresponding SAP Web Client screen
- Templates are fully self-contained single-file HTML — no build step, no external dependencies at runtime
