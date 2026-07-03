---
name: forget
description: Delete one semantic memory by id. Wraps DELETE /v1/memories/{id} scoped per the ingest-gate. Triggered when a memory is wrong or stale — e.g. "/forget mem_abc123", "delete that memory", "that decision was reversed, forget it".
---

# /forget — delete a semantic memory

`DELETE {kernel_url}/v1/memories/{id}?element=<element>`. The one
mutation in the memory trio.

Configuration from `.claude/ingest-gate.md`.

## Behavior contract

- **Requires an explicit `memory_id`.** No argument → run the user's
  description through `/recall` first, show the candidates with ids,
  and let them pick. Never delete on a similarity guess.
- **Invocation with an id is consent** — no extra confirmation prompt.
- **`memory_not_found`** → say so; the id may be stale or in another
  element/scope.
- **Superseded, not wrong?** Suggest `/remember`-ing the replacement
  fact (with the reversal noted) *before* forgetting the old one — a
  corrected record beats a deleted one when the history matters.

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url` + `element`.
2. `DELETE /v1/memories/{id}?element=<element>` → confirm:
   `Forgot <memory_id>.`

## When NOT to use this skill

- **Deleting a document** → `/doc <id> --delete`.
- **Correcting a memory** → `/remember` the correction; forget the old
  one second, if at all.

## What "done" looks like

The memory gone (absent from a subsequent `/recall --list`), the
deletion confirmed by id, nothing else touched.
