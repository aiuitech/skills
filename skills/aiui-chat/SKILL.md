---
name: aiui-chat
description: >-
  Use in whenever the user is in need of a rich web supported message not supported by your own mime type messages such as: text/html, text/markdown (with image rendering), image/*, application/json, etc. Enables POST of data with a mime-type and return of an accessible URL for the user. Only use this skill when the aiui.chat MCP server is not connected and authorized — prefer the MCP tools when available.
---

## Base URL

`https://aiui.chat` — but if `AIUI_URL` is set in your environment, use it as
the base URL instead (it targets a local or staging instance; the MCP server
noted at the bottom lives at that instance's companion MCP origin). The curl
examples below use `${AIUI_URL:-https://aiui.chat}` so they work either way.

## API

```yaml
openapi: 3.1.0
info: { title: aiui.chat, version: "1" }
paths:
  /:
    post:
      summary: Create a message. Body is the raw content; Content-Type is its MIME type.
      parameters:
        - name: filename
          in: query
          schema: { type: string }
          description: Download/display filename for binary blobs.
        - name: session
          in: query
          schema: { type: string }
          description: >-
            Attach to a session you own. Requires auth — 401 without it,
            404 if not yours; never silently ignored.
        - name: children
          in: query
          schema: { type: string }
          description: >-
            Comma-separated ids of messages you created earlier (max 32) to
            adopt as children of this one — use for assets this message embeds.
            Response echoes the adopted subset as `children`.
        - name: x-aiui-agent-session
          in: header
          schema: { type: string }
          description: >-
            Your harness's session/thread id (alias: ?agent_session=). Send it
            whenever you have one — Claude Code hooks expose it as session_id;
            the Agent SDK on init/result messages. Messages sharing an id
            collect into one aiui.chat session (immediately if authenticated,
            at claim time if anonymous). 409 if the id is already bound to
            another user's session.
      requestBody:
        required: true
        content:
          "*/*":
            schema:
              description: Any standard MIME type served as-is, or a vendor type (below).
      responses:
        "201":
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
                  url: { type: string, description: The message URL — share this. }
                  sessionId:
                    type: [string, "null"]
                    description: "null on anonymous creates (assigned when claimed)."
                  children:
                    type: array
                    items: { type: string }
                    description: Ids adopted via ?children= (only present when used).
  /api/messages:
    get:
      summary: >-
        List your messages, newest first (auth required — 401 otherwise). Use
        to find existing message ids/urls to link or embed in new content.
      parameters:
        - { name: session, in: query, schema: { type: string } }
        - name: agent_session
          in: query
          schema: { type: string }
          description: Scope to the session your agent-session id maps to (header alias works too).
        - name: parent
          in: query
          schema: { type: string }
          description: Only children of this message.
        - { name: limit, in: query, schema: { type: integer, default: 50, maximum: 200 } }
      responses:
        "200":
          content:
            application/json:
              schema:
                type: object
                properties:
                  messages:
                    type: array
                    items:
                      type: object
                      properties:
                        id: { type: string }
                        url: { type: string }
                        mime: { type: string }
                        size: { type: integer }
                        filename: { type: [string, "null"] }
                        textPreview: { type: [string, "null"] }
                        sessionId: { type: [string, "null"] }
                        parentId: { type: [string, "null"] }
                        createdAt: { type: string }
```

## Embedding messages in messages

Messages can reference each other by URL: a `text/html` or `text/markdown`
message renders on aiui.chat, so root-relative links to other messages
(`<img src="/MESSAGE_ID">`, `![chart](/MESSAGE_ID)`, `<a href="/MESSAGE_ID">`)
resolve in-page and render for the recipient. Flow:

1. POST each asset (`image/*`, etc.) first; collect the returned ids.
2. Write the page referencing them by `/<id>` (or full URL).
3. POST the page with `?children=<id1>,<id2>` so the assets are organized
   under it (and into its session).

Send the same `x-aiui-agent-session` on every POST so all parts land in one
session. Embedded assets render for whoever owns (or first claims) them — the
same recipient who opens the page — so post page + assets for a single
recipient; anonymous "view without signing in" viewers won't see embedded
assets.

```bash
# HTML page, tagged with this conversation's id (recommended when known)
curl -X POST "${AIUI_URL:-https://aiui.chat}/" -H 'content-type: text/html' \
  -H "x-aiui-agent-session: $YOUR_SESSION_ID" --data-binary @report.html

# Page embedding an image asset (see "Embedding messages in messages")
IMG=$(curl -sX POST "${AIUI_URL:-https://aiui.chat}/" -H 'content-type: image/png' --data-binary @chart.png | jq -r .id)
curl -X POST "${AIUI_URL:-https://aiui.chat}/?children=$IMG" -H 'content-type: text/html' \
  --data-binary "<h1>Report</h1><img src=\"/$IMG\">"

# Interactive map (vendor type)
curl -X POST "${AIUI_URL:-https://aiui.chat}/" -H 'content-type: application/vnd.aiui.chat.map+json' \
  -d '{"title":"Coffee nearby","markers":[{"coordinates":[-122.41,37.77],"label":"Office"}]}'
```

## Lifecycle

- When the first user visits the URL it's claimed
- Unclaimed messages expire 24h after creation
- First viewer decides ownership: signed-in → claimed permanently (private to
  them); anonymous → locked to their browser (still expires within 24h).
  Afterwards the URL is dead for everyone else — treat URLs as single-recipient.
- Every claimed message belongs to a session (one is created automatically if
  none is specified)

## Vendor types

JSON payloads validated against a schema, rendered as an interactive React
view. Invalid payloads still create a message but render the validation error —
check the schema first.

| MIME type                            | Renders as             | Schema                                 |
| ------------------------------------ | ---------------------- | -------------------------------------- |
| `application/vnd.aiui.chat.map+json` | Interactive Mapbox map | [references/map.md](references/map.md) |
| `text/html`                          | A static HTML render   |                                        |

## Authenticated features (MCP)

Claimed-by-default messages, sessions, and push notifications (email,
webhooks): connect to `https://mcp.aiui.chat/mcp` (OAuth with the user's
aiui.chat account).
