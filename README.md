# b1-report-templates

A library of pre-built SAP Business One report templates. Each report HTML file is self-describing: it owns its rendering, metadata, and runtime query definitions. The consuming program uses `manifest.json` only as a lightweight catalog plus shared deeplink definitions.

Hosted via GitHub Pages at:
```
https://<your-username>.github.io/b1-report-templates/manifest.json
```

The consuming program hardcodes only that one URL. Template updates and new reports are available immediately on push.

---

## How it works

```
1. Program fetches manifest.json         → discovers available templates
2. Matches user intent to a template     → by id, tags, or description
3. Fetches the template HTML             → pre-designed, pre-built
4. Reads window.B1Report.meta            → validates template id + query definitions
5. Runs the SL queries from the HTML     → gets live JSON data
6. Calls window.B1Report.load(data)      → template renders itself
7. Delivers the HTML file to the user    → self-contained, no runtime deps
```

The user receives a finished HTML file with charts, tables, and deeplinks into the SAP Web Client.

---

## Repository structure

```
b1-report-templates/
├── manifest.json                ← template catalog + shared deeplinks only
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

Every template must expose `window.B1Report` with this contract:

```js
window.B1Report = {
  meta: {
    id: "my-new-report",
    name: "Human readable name",
    description: "Shown to the AI for intent matching",
    tags: ["keyword1", "keyword2"],
    queries: [
      {
        key: "primaryData",
        endpoint: "Invoices",
        select: "DocEntry,DocNum,CardCode,CardName,DocDate,DocTotal",
        filter: "DocDate ge '{date_6m_ago}' and Cancelled eq 'tNO'",
        orderby: "DocDate desc",
        top: 200
      },
      {
        key: "summary",
        sql: "SELECT T0.\"CardCode\", SUM(T0.\"DocTotal\") AS \"Total\" FROM OINV T0 WHERE T0.\"CANCELED\" = 'N' GROUP BY T0.\"CardCode\""
      }
    ]
  },

  load(data) {
    // data.primaryData = rows from the first query
    // data.summary     = rows from the second query
    // data.company
    // data.webClientBase
    // data.deeplinks
  }
};
```

Rules:

- `meta.id` must match the HTML filename and the manifest report id.
- `meta.queries` must contain full query objects.
- String references such as `queries: ["rows"]` are invalid.
- Each query must define either `endpoint` or `sql`.

### 3. Add demo mode

Keep a realistic standalone demo payload so the template can be opened directly in a browser during design work.

### 4. Build SAP Web Client deeplinks

Use the injected `buildObjectUrl` helper together with shared deeplink definitions from `data.deeplinks`:

```js
const sapLink = (type, value) => {
  const def = deeplinks[type];
  if (!def || !value) return "";
  return B1Report.buildObjectUrl(def.objName, value);
};
```

### 5. Register in manifest.json

Add only catalog metadata plus any shared deeplink definitions:

```json
{
  "id": "my-new-report",
  "type": "standalone",
  "name": "My new report",
  "description": "What this report shows",
  "tags": ["keyword1", "keyword2"]
}
```

Do not put report queries in the manifest. The HTML file is the runtime source of truth.

### 6. Push

```bash
git add my-new-report.html manifest.json
git commit -m "add my new report"
git push
```

---

## manifest.json field reference

```jsonc
{
  "version": 2,
  "updated": "2026-04-15",
  "deeplinks": {
    "invoice": { "objName": "Invoices", "key": "DocEntry" }
  },
  "reports": [
    {
      "id": "string",
      "type": "standalone",
      "name": "string",
      "description": "string",
      "tags": ["string"]
    }
  ]
}
```

### Query field reference

Runtime query definitions live in `window.B1Report.meta.queries` inside each HTML template.

```jsonc
[
  {
    "key": "string",
    "endpoint": "Invoices",
    "select": "...",
    "filter": "...",
    "orderby": "...",
    "top": 100
  },
  {
    "key": "string",
    "sql": "SELECT ..."
  }
]
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

- arithmetic inside aggregate functions
- arithmetic between aggregates in `SELECT`
- `ORDER BY` after `GROUP BY`

Return raw aggregates from SQL and compute derived values in `load()` when needed.
