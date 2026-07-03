---
name: doc
description: Show one document record in full — status, version, pages, chunks, extraction stats, stored error if failed — or delete it with --delete (cascade removes record + chunks + vectors; the honest counts are echoed). Wraps GET/DELETE /v1/documents/{id}. Triggered when the user wants one record — e.g. "/doc abc123", "/doc abc123 --delete", "show that document's record", "remove that document from the corpus".
---

# /doc — show or delete one document record

`GET {kernel_url}/v1/documents/{id}?element=<element>` to show;
`DELETE` (with `--delete`) to remove. Deletion cascades kernel-side —
record + every chunk + every vector — and the response reports honest
counts; this skill echoes them verbatim.

Configuration (kernel URL, element) comes from `.claude/ingest-gate.md`.

## Behavior contract

- **Show is read-only; delete is the one mutation in this skill.**
  `--delete` is the consent — invoked with the flag, delete without an
  extra "are you sure?". Invoked without it, never delete.
- **Echo the cascade honestly.** Render the kernel's
  `{deleted, deleted_chunks}` counts as returned. If the chunk count
  looks surprising (0 chunks deleted for a document that listed 40),
  say so — that's a signal, not noise (see ingest-rules.md →
  persistence window).
- **`document_not_found`** → say so and suggest `/docs`; the id may
  belong to another element. Don't retry against other elements
  unprompted.

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url` + `element`.
2. **Show** (default): `GET /v1/documents/{id}?element=<element>` →
   render the full record: name, doc_id, status, version, namespace,
   pages, chunks, extraction stats, timestamps — and the stored `error`
   prominently if status is `failed`.
3. **Delete** (`--delete`): `DELETE /v1/documents/{id}?element=<element>`
   → render: `Deleted <name> (<id>): record + <deleted_chunks> chunks/vectors.`
4. If the project keeps an audit ledger (tasks module or its own),
   suggest recording a deletion of anything substantial — soft
   convention, not a gate.

## When NOT to use this skill

- **Listing everything** → `/docs`.
- **Updating a document's content** → `/reingest` (delete + re-ingest
  loses the version history; reingest bumps it).
- **Deleting a memory** → `/forget`.

## What "done" looks like

Show: the record rendered in full, failures visible. Delete: the
kernel's cascade counts echoed verbatim, and the record gone from a
subsequent `/docs`.
