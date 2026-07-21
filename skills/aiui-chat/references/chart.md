# application/vnd.aiui.chat.chart+json

A chart (line, bar, area, pie) rendered from inline series data with tooltips
and a legend. Rows live directly in the message — no separate data message.

## JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "data", "xKey", "series"],
  "properties": {
    "title": { "type": "string", "description": "Heading shown above the chart" },
    "description": { "type": "string", "description": "Subtext shown under the title" },
    "type": { "enum": ["line", "bar", "area", "pie"], "description": "Chart form" },
    "data": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": {
          "anyOf": [{ "type": "string" }, { "type": "number" }, { "type": "null" }]
        }
      },
      "description": "Rows of data; keys referenced by xKey and series[].key. Series values must be numbers — null/missing points are skipped"
    },
    "xKey": {
      "type": "string",
      "description": "Row key for the x axis (or pie slice label)"
    },
    "series": {
      "type": "array",
      "items": { "$ref": "#/$defs/series" },
      "description": "One entry per plotted series (pie uses only the first)"
    },
    "stacked": {
      "type": "boolean",
      "description": "Stack series on top of each other (bar and area only)"
    }
  },
  "$defs": {
    "series": {
      "type": "object",
      "required": ["key"],
      "properties": {
        "key": { "type": "string", "description": "Data key this series reads from each row" },
        "label": { "type": "string", "description": "Legend/tooltip label; defaults to key" },
        "color": {
          "type": "string",
          "description": "CSS color; defaults to the theme chart palette by position"
        }
      }
    }
  }
}
```

## Example

```json
{
  "title": "Monthly active users",
  "description": "Web vs mobile, Q1",
  "type": "line",
  "xKey": "month",
  "data": [
    { "month": "Jan", "web": 1200, "mobile": 900 },
    { "month": "Feb", "web": 1350, "mobile": 1100 },
    { "month": "Mar", "web": 1600, "mobile": 1500 }
  ],
  "series": [
    { "key": "web", "label": "Web" },
    { "key": "mobile", "label": "Mobile" }
  ]
}
```

## Authoring guidance

- **Pick the form for the job**: `line`/`area` for change over time, `bar` for
  comparing categories, `pie` only for a handful of parts-of-a-whole slices.
- **Omit `series[].color`** unless a color is meaningful (e.g. matching an
  established convention). The default palette is theme-aware (light/dark) and
  colorblind-safe; hand-picked colors are neither, and a custom color is used
  as-is in both themes.
- Keep it to **at most 5 series** — the palette has five distinguishable
  slots, and more than that is unreadable anyway. Fold the rest into an
  "Other" series.
- `stacked: true` only affects `bar` and `area`. Use it for parts-of-a-whole
  over a dimension; leave it off to compare series against each other.
- **Pie** uses `xKey` as the slice label and the first series as the value;
  extra series are ignored.
- A legend appears automatically when more than one series is plotted (or per
  slice for pie). Values are formatted compactly (12.3K, 1.2M) — pre-scale
  your numbers only if the units need it, and put units in `label` (e.g.
  `"Revenue (USD)"`).
- Rows missing a series key (or with `null`) are simply skipped for that
  series; a series with no numeric values at all is dropped with a note. An
  empty `data` array renders a "No data" note, not an error.
