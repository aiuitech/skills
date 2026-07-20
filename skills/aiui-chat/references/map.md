# application/vnd.aiui.chat.map+json

Interactive Mapbox map rendered at the message URL. All coordinates are
`[longitude, latitude]` (GeoJSON order), not lat/lng.

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "title": { "type": "string", "description": "Heading shown above the map" },
    "center": {
      "$ref": "#/$defs/lngLat",
      "description": "Initial center; defaults to fitting all markers"
    },
    "zoom": {
      "type": "number",
      "minimum": 0,
      "maximum": 22,
      "description": "Defaults to fitting all markers"
    },
    "style": { "enum": ["streets", "outdoors", "light", "dark", "satellite"] },
    "markers": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["coordinates"],
        "properties": {
          "coordinates": { "$ref": "#/$defs/lngLat" },
          "label": { "type": "string", "description": "Short marker label" },
          "description": { "type": "string", "description": "Popup text" },
          "color": { "type": "string", "description": "CSS color, e.g. #e11d48" }
        }
      }
    }
  },
  "$defs": {
    "lngLat": {
      "type": "array",
      "prefixItems": [
        { "type": "number", "minimum": -180, "maximum": 180 },
        { "type": "number", "minimum": -90, "maximum": 90 }
      ],
      "minItems": 2,
      "maxItems": 2
    }
  }
}
```

## Example

```json
{
  "title": "Coffee near the office",
  "center": [-122.4194, 37.7749],
  "zoom": 13,
  "style": "streets",
  "markers": [
    { "coordinates": [-122.4194, 37.7749], "label": "Office" },
    {
      "coordinates": [-122.4077, 37.7857],
      "label": "Sightglass Coffee",
      "description": "Great pour-over, opens 7am",
      "color": "#e11d48"
    }
  ]
}
```
