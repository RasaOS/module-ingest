---
name: recall
description: Similarity-search the project's semantic memories and render content · distance · memory_id. Wraps POST /v1/memories/query scoped per the ingest-gate. Triggered when the user wants to look something up from memory — e.g. "/recall what did we decide about indexes?", "did we make a call on X?", "search memory for Y".
---

# /recall — search semantic memories

`POST {kernel_url}/v1/memories/query` with the query text, scoped to
the gate's element (+ `memory_scope` unless the user widens it), and
render the closest memories with distances. Read-only.

Configuration from `.claude/ingest-gate.md`.

## Behavior contract

- **Body**: `{"query": "<text>", "element": "<gate element>",
  "scopes": ["<gate memory_scope>"]}`. `--k <n>` caps at 25.
  `--all-scopes` drops the scopes filter. Omit unset fields.
- **Render each hit**: content · `d=<distance>` · `memory_id`. Distance
  is the honesty signal — a best hit at d=0.6 is a weak match; say so
  rather than presenting it as the answer.
- **Empty result is an answer**: "nothing remembered about that" +
  point at `/remember`. Never pad with guesses.
- **List mode** (`/recall --list`): `GET /v1/memories?scope=&element=`
  (newest first) instead of a similarity query — for "what do we have
  remembered?" rather than "find X".

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url`, `element`,
   `memory_scope`.
2. Query (or list); render hits with content, distance, memory_id.
3. Close with one line: `N memories, best d=<distance>.`

## When NOT to use this skill

- **Searching documents** → `/doc-query`.
- **Deleting** → `/forget <memory_id>` (this skill surfaces the ids).
- **Mid-conversation with `memory_search` available** — just use the
  tool.

## What "done" looks like

The closest memories rendered with distances and ids, weak matches
labeled as weak, and nothing mutated.
