---
name: aiui-chat
description: Create rich, shareable web messages on aiui.chat with a single HTTP POST. Supports standard MIME types (text/html, text/markdown, application/json, image/png, application/pdf, ...) and rich vendor types like application/vnd.aiui.chat.map+json (interactive Mapbox maps). Use when the user wants output delivered as a URL, wants to share rich content (pages, maps, images, documents) beyond the terminal, or needs asynchronous cross-channel communication.
---

# aiui.chat — rich messages over a URL

aiui.chat is a web message bus for agents. You POST content with a MIME type;
it returns a short URL that renders that content on the web. No API key or
authentication is required to create a message.

## Creating a message

```
POST https://aiui.chat/
Content-Type: <mime type of the body>

<raw body: text, JSON, or binary>
```

Response (201):

```json
{ "id": "fh1rdgczrt", "url": "https://aiui.chat/fh1rdgczrt" }
```

Optional query parameters:

- `?filename=report.pdf` — sets the download/display filename for binary blobs.
- `?session=<sessionId>` — attaches the message to one of the user's sessions
  (only honored on authenticated requests, e.g. via the MCP server).

Examples:

```bash
# An HTML page
curl -X POST https://aiui.chat/ -H 'content-type: text/html' \
  --data-binary @report.html

# An image
curl -X POST "https://aiui.chat/?filename=chart.png" \
  -H 'content-type: image/png' --data-binary @chart.png

# An interactive map (vendor type — see below)
curl -X POST https://aiui.chat/ \
  -H 'content-type: application/vnd.aiui.chat.map+json' \
  -d '{"title":"Coffee nearby","markers":[{"coordinates":[-122.41,37.77],"label":"Office"}]}'
```

Always give the returned `url` to the user — that URL is the message.

## Message lifecycle (important)

- A new message is **unclaimed and expires 24 hours after creation** if never
  viewed.
- The **first viewer decides ownership**: a signed-in viewer claims it
  permanently (only they can see it afterwards); an anonymous viewer can lock
  it to their browser instead (it still expires within the original 24h).
- After a message is claimed or anonymously locked, the URL is dead for
  everyone else. Treat URLs as single-recipient.
- Messages created through the authenticated MCP server are born claimed by
  the user and never expire.

## Message types

### Standard MIME types

Any standard type is accepted and served as-is at the URL: `text/plain`,
`text/html` (rendered as a real page), `text/markdown`, `application/json`,
`image/png`, `image/jpeg`, `image/svg+xml`, `application/pdf`, audio/video
types, etc. Send the raw bytes with the right `Content-Type`.

### Rich vendor types

Vendor types are JSON payloads validated against a schema and rendered as an
interactive React view at the URL.

| MIME type                            | Renders as                                          | Schema reference                       |
| ------------------------------------ | --------------------------------------------------- | -------------------------------------- |
| `application/vnd.aiui.chat.map+json` | Interactive Mapbox map with markers, popups, styles | [references/map.md](references/map.md) |

Minimal map example:

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

All coordinates are `[longitude, latitude]`. Invalid payloads still create a
message, but the URL shows the validation error instead of a map — validate
against the schema reference before sending.

## Authenticated features (MCP)

For claimed-by-default messages, sessions (organized conversations), and
push notifications across channels (email, webhooks), connect to the MCP
server at `https://mcp.aiui.chat/mcp` (OAuth — the user signs in with their
aiui.chat account).
