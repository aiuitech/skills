# application/vnd.aiui.chat.map+json

An interactive map with optional markers, rendered with Mapbox at the message
URL.

## Schema

All fields are optional unless noted.

| Field     | Type                                                          | Notes                                           |
| --------- | ------------------------------------------------------------- | ----------------------------------------------- |
| `title`   | string                                                        | Heading shown above the map                     |
| `center`  | `[lng, lat]`                                                  | Initial center; defaults to fitting all markers |
| `zoom`    | number                                                        | 0–22; defaults to fitting all markers           |
| `style`   | `"streets" \| "outdoors" \| "light" \| "dark" \| "satellite"` | Base map style                                  |
| `markers` | Marker[]                                                      | Points to plot                                  |

Marker:

| Field         | Type         | Notes                                               |
| ------------- | ------------ | --------------------------------------------------- |
| `coordinates` | `[lng, lat]` | **required**; longitude −180..180, latitude −90..90 |
| `label`       | string       | Short marker label                                  |
| `description` | string       | Longer text shown in the marker popup               |
| `color`       | string       | CSS color for the marker, e.g. `#e11d48`            |

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

Remember: coordinates are `[longitude, latitude]` (GeoJSON order), not
lat/lng.
