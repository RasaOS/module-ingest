---
name: remember
description: Store a semantic memory in the project's element — a decision, preference, or durable fact, embedded for similarity recall. Scope comes from the ingest-gate; supports --ttl for facts with an expiry. Wraps POST /v1/memories. Triggered when something worth recalling later surfaces — e.g. "/remember we chose per-element indexes", "remember that the client prefers …", "store this decision".
---

# /remember — store a semantic memory

`POST {kernel_url}/v1/memories` with the text, scoped to the gate's
element + `memory_scope`. Memories are embedded and recalled by
**similarity** (`/recall`, or the injected `memory_search` tool) — this
is the long-term semantic store, not the exact-recall KV
(`/v1/memory/{namespace}`).

Configuration from `.claude/ingest-gate.md`; the documents-vs-memory
boundary from `.claude/ingest-rules.md`.

## Behavior contract

- **Memory = one self-contained statement.** Write it to be found by a
  future similarity search: include the *what* and the *why*, stamp
  relative dates absolute ("chose X over Y, 2026-07-02"), keep it under
  16KB (the cap — but if you're anywhere near it, it's a document;
  hand off to `/ingest`).
- **Body**: `{"content": "<text>", "scope": "<gate memory_scope>",
  "element": "<gate element>"}`. `--ttl <duration>` → `ttl_s` in
  seconds (accept `90d` / `12h` style and convert). Omit fields you
  don't set — never `""`.
- **Echo the stored memory** — `memory_id` + scope — so `/forget` has
  a handle.

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url`, `element`,
   `memory_scope`.
2. If the input is multi-paragraph or file-shaped, push back once:
   suggest `/ingest` (documents) or a tightened one-liner (memory).
3. `POST /v1/memories` → render:
   `Remembered (<memory_id>, scope <scope>): <content>` (+ TTL if set).

## When NOT to use this skill

- **Corpora / files** → `/ingest`.
- **Exact-recall config values** → the KV memory surface, not the
  semantic store.
- **Mid-conversation with the `memory_store` tool available** — just
  use the tool.

## What "done" looks like

One memory stored, its id echoed, findable by a `/recall` of a
paraphrase of it.
