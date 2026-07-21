# application/vnd.aiui.chat.diff+json

A code diff rendered from unified patch text: per-file sections with paths and
+N/−N counts, hunk headers, old/new line-number gutters, added/removed line
coloring, and syntax highlighting inferred from each file's extension. The
viewer can toggle between unified and side-by-side layouts.

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["diff"],
  "properties": {
    "title": { "type": "string", "description": "Heading shown above the diff" },
    "description": {
      "type": "string",
      "description": "Context shown under the title (markdown-free)"
    },
    "diff": {
      "type": "string",
      "description": "Unified diff text — paste `git diff` / `diff -u` output verbatim; may span multiple files"
    },
    "view": {
      "enum": ["unified", "split"],
      "description": "Initial layout (viewer can toggle); defaults to unified"
    }
  }
}
```

## Example

```json
{
  "title": "Fix retry backoff",
  "description": "Exponential instead of linear backoff, capped at 30s.",
  "diff": "diff --git a/src/retry.ts b/src/retry.ts\nindex 6b3f1a2..9c4e8d1 100644\n--- a/src/retry.ts\n+++ b/src/retry.ts\n@@ -1,4 +1,4 @@\n export const backoff = (attempt: number) =>\n-  Math.min(1000 * attempt, 30_000);\n+  Math.min(1000 * 2 ** attempt, 30_000);\n"
}
```

## Guidance

- Paste `git diff` / `git show` / `diff -u` output **verbatim** into `diff` —
  don't strip the `diff --git`, `index …`, mode, or rename lines; the parser
  tolerates all of them (and also accepts diffs that start straight at
  `--- / +++` or a bare `@@` hunk).
- Multi-file diffs work: each file renders as its own section with the path
  (`old → new` on renames), added/removed counts, and its hunks.
- Syntax highlighting is chosen from the file extension (ts, py, go, rs, css,
  …); unknown extensions render uncolored, which is fine.
- `view: "split"` starts side-by-side — good for reviewing substantial
  rewrites on wide screens. Omit it (unified) for small patches; the viewer
  can always toggle.
- If the text can't be parsed as a unified diff it is shown raw with a
  warning rather than erroring — but prefer sending real patch text.
- Keep patches to what matters: trim unrelated files before posting rather
  than sending a whole repository diff.
