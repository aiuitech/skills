# application/vnd.aiui.chat.table+json

An interactive data table (per-column sorting, global search, optional
per-column filters and pagination). The config message holds no rows — the
data lives in a separate `text/csv` CHILD message referenced by `source`.

Two-step flow, always in this order:

1. POST the CSV as its own `text/csv` message; note the returned `id`.
2. POST the table config with `?children=<csv id>` (HTTP) or
   `children: [id]` (MCP `create_message`) and `"source": "<csv id>"`.

The `children` adoption keeps the pair attached: the CSV moves into the same
session, shares the config's owner, and inherits visibility when the config
is published — so the rendered table can always fetch its rows.

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["source"],
  "properties": {
    "title": { "type": "string", "description": "Heading shown above the table" },
    "source": {
      "type": "string",
      "description": "Message id of the text/csv child message holding the row data"
    },
    "columns": {
      "type": "array",
      "description": "Columns to show, in order; defaults to every CSV header as a string column",
      "items": {
        "type": "object",
        "required": ["key"],
        "properties": {
          "key": { "type": "string", "description": "CSV header this column reads" },
          "label": { "type": "string", "description": "Display label; defaults to key" },
          "type": {
            "enum": ["string", "number", "date"],
            "description": "number sorts numerically and right-aligns; date sorts by parsed date but renders the raw text; defaults to string"
          },
          "filterable": {
            "type": "boolean",
            "description": "Show a per-column substring filter input under the header (default false)"
          }
        }
      }
    },
    "search": {
      "type": "boolean",
      "description": "Show a global search box over all columns (default true)"
    },
    "pageSize": {
      "type": "number",
      "description": "Rows per page; omit to show every row with no pager"
    },
    "initialSort": {
      "type": "object",
      "required": ["key"],
      "properties": {
        "key": { "type": "string", "description": "Column key to sort by initially" },
        "desc": { "type": "boolean", "description": "Sort descending (default ascending)" }
      }
    }
  }
}
```

## Example

```bash
CSV=$(curl -sX POST "${AIUI_URL:-https://aiui.chat}/" -H 'content-type: text/csv' \
  --data-binary $'invoice,amount,due\nINV-1,1200,2026-08-01\nINV-2,860,2026-08-15' | jq -r .id)
curl -X POST "${AIUI_URL:-https://aiui.chat}/?children=$CSV" \
  -H 'content-type: application/vnd.aiui.chat.table+json' \
  -d "{
    \"title\": \"Q2 invoices\",
    \"source\": \"$CSV\",
    \"columns\": [
      { \"key\": \"invoice\", \"label\": \"Invoice #\", \"filterable\": true },
      { \"key\": \"amount\", \"label\": \"Amount\", \"type\": \"number\" },
      { \"key\": \"due\", \"label\": \"Due date\", \"type\": \"date\" }
    ],
    \"pageSize\": 25,
    \"initialSort\": { \"key\": \"due\" }
  }"
```

Share the CONFIG message's URL — the CSV child is an implementation detail.

## Authoring guidance

- CSV must be RFC 4180: first record is the header row; quote fields that
  contain commas, quotes, or newlines; escape quotes by doubling (`""`).
  Comma delimiter only.
- Column `key` must match a CSV header exactly (case-sensitive). Unknown keys
  render as an empty column — double-check spelling.
- Omit `columns` entirely for a quick table: every header becomes a sortable
  string column.
- `type: "number"` — sorting strips `$`, `%`, commas, and whitespace before
  comparing, so `$1,200` sorts fine; unparseable values group at the low end.
- `type: "date"` — sorted via `Date.parse`, so use ISO `YYYY-MM-DD` (or any
  Date-parseable format); the cell shows your raw text unchanged.
- Set `pageSize` for anything beyond a few dozen rows; leave `search` on
  unless the table is tiny. Use `filterable` sparingly — one or two key
  columns — since it adds a filter row under the header.
- To update the data, post a new CSV + new config; messages are immutable.
