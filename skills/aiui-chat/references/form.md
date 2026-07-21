# application/vnd.aiui.chat.form+json

A form the user fills in at the message URL. When they submit, the web app
creates an `application/vnd.aiui.chat.form-response+json` message as a CHILD
of the form message (`parentId` = form id, same session) — the pair stays
related, and you read the structured answers by polling for children.

One response per form: after the first submission the form renders the
submitted answers read-only, and further submits are rejected (409). Post a
new form message if you need to ask again.

## JSON Schema (form)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["fields"],
  "properties": {
    "title": { "type": "string", "description": "Form heading" },
    "description": { "type": "string", "description": "Intro text shown above the fields" },
    "fields": {
      "type": "array",
      "minItems": 1,
      "description": "Fields, in order. Names must be unique.",
      "items": { "$ref": "#/$defs/field" }
    },
    "submitLabel": { "type": "string", "description": "Submit button text; defaults to Submit" }
  },
  "$defs": {
    "field": {
      "type": "object",
      "required": ["name", "type"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "description": "Answer key in the response payload"
        },
        "label": { "type": "string", "description": "Field label; defaults to name" },
        "type": { "enum": ["text", "textarea", "number", "select", "multiselect", "boolean"] },
        "required": {
          "type": "boolean",
          "description": "User must answer before submitting. No effect on boolean fields — they always submit true or false."
        },
        "placeholder": { "type": "string" },
        "description": { "type": "string", "description": "Help text under the field" },
        "options": {
          "type": "array",
          "minItems": 1,
          "description": "Choices — required for select/multiselect, ignored otherwise",
          "items": {
            "type": "object",
            "required": ["value"],
            "properties": {
              "value": {
                "type": "string",
                "minLength": 1,
                "description": "Submitted value for this choice"
              },
              "label": { "type": "string", "description": "Display label; defaults to value" }
            }
          }
        }
      }
    }
  }
}
```

## Example (form)

```json
{
  "title": "Deploy approval",
  "description": "v2.4.0 is ready for production.",
  "fields": [
    {
      "name": "decision",
      "label": "Ship it?",
      "type": "select",
      "required": true,
      "options": [{ "value": "approve" }, { "value": "hold" }]
    },
    { "name": "notes", "label": "Notes", "type": "textarea", "placeholder": "Anything to flag?" }
  ],
  "submitLabel": "Send decision"
}
```

```bash
curl -X POST "${AIUI_URL:-https://aiui.chat}/" \
  -H 'content-type: application/vnd.aiui.chat.form+json' \
  -H "x-aiui-agent-session: $YOUR_SESSION_ID" \
  --data-binary @form.json
```

# application/vnd.aiui.chat.form-response+json

The user's submitted answers, created by the web app as a child of the form
message. You read these; you rarely create them. Answer values are validated
against the parent form: unknown field names are rejected, required fields
must be present, select/multiselect values must be declared options, and
value types must match the field type.

| Field type        | Answer value type | Notes                                    |
| ----------------- | ----------------- | ---------------------------------------- |
| `text`/`textarea` | `string`          | Empty optional answers are omitted       |
| `number`          | `number`          | Omitted when optional and left blank     |
| `select`          | `string`          | One of the field's option `value`s       |
| `multiselect`     | `string[]`        | Each entry one of the option `value`s    |
| `boolean`         | `boolean`         | Always present — unchecked means `false` |

## JSON Schema (form-response)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["answers"],
  "properties": {
    "answers": {
      "type": "object",
      "description": "Field name → submitted value",
      "additionalProperties": {
        "oneOf": [
          { "type": "string" },
          { "type": "number" },
          { "type": "boolean" },
          { "type": "array", "items": { "type": "string" } }
        ]
      }
    }
  }
}
```

## Example (form-response)

```json
{ "answers": { "decision": "approve", "notes": "LGTM — watch error rates after rollout." } }
```

## Reading the reply

The submission lands as a new message in the form's session, so it arrives
through the same channels as any other reply:

- **MCP (preferred):** call `check_messages` with `since` = the newest
  timestamp you've seen and `wait=25` — the call parks server-side and returns
  the moment the form-response lands (its `parentId` is the form's id). Read
  the answers with `get_message`.
- **HTTP (authenticated):** list the form's children and fetch the response
  body:

  ```bash
  curl -s "${AIUI_URL:-https://aiui.chat}/api/messages?parent=$FORM_ID"
  # → messages[] — the form-response child has mime application/vnd.aiui.chat.form-response+json
  curl -s "${AIUI_URL:-https://aiui.chat}/$RESPONSE_ID/raw"
  # → { "answers": { ... } }
  ```

  `GET /api/messages` requires auth (401 otherwise) — anonymous, skill-only
  agents can't poll for the reply over HTTP; connect the MCP server to read
  answers.

Submission itself is browser-only: the signed-in owner posts to
`POST /api/messages/<form id>/response`, which validates the answers and
creates the child through the normal pipeline (notifications fire, MCP
long-polls wake). Agents never call it.
