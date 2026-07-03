---
name: docs
description: List the document records in this project's element — name, status, version, pages, chunks, updated — flagging failed rows with their stored error. Read-only view over GET /v1/documents. Triggered when the user wants to see what's in the corpus — e.g. "/docs", "what documents are ingested?", "list the corpus", "show the document store".
---

# /docs — list document records

Read + render the element's document records from
`GET {kernel_url}/v1/documents?element=<element>`. The read-side
companion to `/ingest`. **Never mutates** anything.

Configuration (kernel URL, element) comes from `.claude/ingest-gate.md`.
An explicit argument (`/docs <element>`) overrides the gate's element —
useful for peeking at another element's corpus or `global`.

## Behavior contract

- **Read-only.** To delete, hand off to `/doc <id> --delete`; to add,
  `/ingest`.
- **Read live state.** Always query the kernel; never render from
  memory of a previous listing.
- **Flag failures.** Rows with `status: failed` render with their
  stored `error` — a failed ingest that looks like a listed document is
  a lie.

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url` + `element` (or use
   the argument).
2. `curl -sS "{kernel_url}/v1/documents?element=<element>"`.
3. Render a table: **name · status · version · pages · chunks ·
   updated**. Failed rows get their error inline. Close with one line:
   `N documents (M failed) in <element>.`
4. Empty store → say so plainly and point at `/ingest`. Don't render an
   empty table frame.

## When NOT to use this skill

- **One record in detail** → `/doc <id>`.
- **Searching content** → `/doc-query` (this lists records, it doesn't
  search inside them).
- **Memories** → `/recall` (memories are namespaced separately).

## What "done" looks like

A rendered table matching the kernel's live state, failures visible,
and nothing on disk or in the kernel changed.
