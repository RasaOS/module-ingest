# Ingest Rules

The portable ingest spine that `rasa.module.ingest` installs at
`.claude/ingest-rules.md`. It covers when to ingest, the documents-vs-
memory boundary, namespace conventions, how to drive the kernel's
ingest/RAG/memory surface, and the operational gotchas. **Read this file
when ingesting documents, querying the corpus, or working with semantic
memory.**

This file is **Element-owned** — it refreshes on upgrade. It deliberately
does **not** decide the things that vary per project:

1. **Which kernel** (URL + auth),
2. **Which element identity** every call is scoped to, and
3. **Which paths `/ingest` may sweep** — and which it must refuse.

All three live in the project-owned **`.claude/ingest-gate.md`** (the
ingest analogue of the task module's done-gate). This file references the
gate; it hardcodes none of them. Never assume `:3001` — read the gate.

## The engine is the kernel's — don't rebuild it

The kernel owns the whole pipeline (canon SA-029): file staging
(`POST /v1/files`), ingest with magic-byte format detection
(`POST /v1/documents`), per-element vector indexes, RAG query
(`POST /v1/documents/query`), semantic memory (`/v1/memories`), and SSE
progress streams. Skills in this module drive that HTTP surface; they do
not re-implement extraction, chunking, or embedding, and neither should
you.

**Supported formats** (detected by magic bytes — the extension is a
claim, not a fact): pdf (text-first; vision fallback for scanned pages
when the kernel has an Anthropic API key + poppler), docx, xlsx
(sheet-per-page), csv, json + jsonl, text, markdown. Anything else fails
honestly with `unsupported_mime`.

## You already have the tools — use them

Every claude turn in a kernel-driven session gets the `kmem` MCP tools
injected automatically: **`doc_search`**, **`memory_search`**, and
**`memory_store`**, already scoped to this element. The skills below are
the *human* workflow; the tools are *yours*.

- When asked about content that may live in the ingested corpus —
  a contract clause, a spec detail, a number from a spreadsheet —
  **run `doc_search` before saying you don't know.** Citing an ingested
  document by name and page beats reconstructing from memory.
- When a decision, preference, or durable fact surfaces in conversation,
  **store it with `memory_store`** so future sessions recall it.
- At the start of work on a long-running concern, a quick
  `memory_search` for prior decisions is cheaper than re-deriving them.

If `RASA_MEMORY=off` the tools are absent; fall back to the skills'
HTTP calls.

## When to ingest vs read directly

Ingest is for **corpora you will query repeatedly by meaning** — not a
replacement for reading a file.

- **Read directly** (Read tool / cat): a file you need once, a file you
  are editing, anything already in the working tree that fits in
  context. Ingesting a file you could just read adds latency and
  storage for nothing.
- **Ingest**: reference material too large for context (books, contract
  sets, specs, exports), material you'll cite across many sessions, and
  binary formats whose value is semantic lookup (a 40-sheet xlsx, a
  200-page PDF).
- **Never ingest**: secrets, credentials, or anything matching the
  gate's `excludes`. Vectors are queryable by anyone with element
  access — an ingested `.env` is a leaked `.env`.

## Documents vs memory — the boundary

- **Memory** = decisions and facts. Single self-contained statements,
  ≤16KB, written to be recalled by similarity: "We chose per-element
  vector indexes over one global index (2026-07-01)." Use `/remember`
  or `memory_store`. Supports TTL for facts with an expiry.
- **Documents** = corpora. Files with pages, structure, and provenance,
  queried with citations. Use `/ingest`.

If you're about to `/remember` three paragraphs, it's probably a
document. If you're about to `/ingest` one sentence, it's a memory.

## Scoping model

- **element** — THE knowledge unit. Every document, memory, and query is
  scoped to one element's index. The gate's `element:` line names this
  project's identity; `global` is the element-less default. Vector
  dimensions are per-element (see dims_mismatch below).
- **namespace** — within an element: `docs` (default for documents),
  `memory.<scope>` (where memories live), and anything you invent
  (`docs.contracts`, `events`, …). Query tiering: `docs*` / `memory*`
  rank tier 1, `events*` tier 2, everything else tier 3.
- **An unfiltered query spans ALL namespaces in the element** — memories
  will surface next to document chunks. That is by design; when a skill
  wants documents only, it filters with `namespaces: ["docs"]`.

## Driving the HTTP surface

Kernel URL + auth come from `.claude/ingest-gate.md` (dev mode: no JWT).
The canonical flows:

- **Ingest a file**: `POST {kernel_url}/v1/files` (multipart, `file`
  field) → `{file_id}` → `POST {kernel_url}/v1/documents` with
  `{file_id, name, element, namespace?}` → 202 `{task_id, doc_id,
  stream_url}` → follow `GET /v1/commands/{task_id}/stream` (plain
  `text/event-stream`; `curl -N` + line filtering is enough — standard
  `started`/`text`/`result`/`end` events, no special event types).
- **Ingest raw text**: same `POST /v1/documents` with `{text, name, …}`
  instead of `file_id`.
- **Re-ingest / update**: `POST /v1/documents` with the existing
  `doc_id` — the kernel bumps the version and replaces chunks
  atomically.
- **Query**: `POST /v1/documents/query` with `{prompt, element,
  namespaces?, k?, organize? flat|grouped|context, window?,
  token_budget?, since_ts?}`.
- **Memory**: `POST /v1/memories` `{content, scope, element, ttl_s?}`;
  `POST /v1/memories/query` `{query, scopes?, k?, element}`;
  `GET /v1/memories?scope=&element=`; `DELETE /v1/memories/{id}`.

**Omit absent optional fields — never send `""`.** An empty string is
not "unset"; building bodies with `"namespace": ""` is the known
empty-string bug class. Construct JSON with only the fields you mean.

## Error codes (handle, don't guess)

| Code | Meaning | Response |
|---|---|---|
| `document_not_found` | No record with that id in this element | Check `/docs`; the id may belong to another element |
| `memory_not_found` | No memory with that id | Check `/recall` / list output |
| `ingest_failed` | Extraction/embedding failed; error stored on the record | Show the record's `error`; don't retry blind |
| `unsupported_mime` | Format not in the supported set | Say so; don't transcode and sneak it in |
| `embedding_provider_unavailable` | `RASA_EMBEDDINGS=openai` with no key (auto/local never hit this) | Tell the user to fix the key or switch to auto/local |
| `dims_mismatch` | Element has vectors from a different embedding model | Re-ingest the element's corpus (see below) |
| `bad_request` | Malformed body (check for `""` fields, caps) | Fix the body |
| `rate_limited` | Back off | Wait and retry once, then surface |

## Caps (surface these, don't silently truncate)

Layout body ≤1MB · memory content ≤16KB · ≤2000 chunks/doc ·
`k` ≤50 (documents) / ≤25 (memories) · tables ≤20k rows ·
JSON ≤50k leaves. A request past a cap fails honestly — report the cap,
don't shave the input without saying so.

## Known operational gotchas

1. **Persistence window.** The kernel's state store runs
   `appendonly no`, `save 60 1000` — a kernel-container restart can
   silently lose recent vectors/documents (a kernel fix — AOF — is
   flagged, not landed). If counts look wrong after a kernel restart,
   **re-ingest** — and say that's what happened rather than papering
   over it.
2. **Vision fallback prerequisites.** Scanned-PDF pages need the kernel
   to have an `ANTHROPIC_API_KEY` (API key, not OAuth) — poppler is
   baked in. Without it, scanned pages come back `low_yield` — flagged,
   never silently empty. `/ingest` warns (does not fail) on
   `low_yield_pages > 0`.
3. **Embedding switching = re-ingest.** Embeddings are zero-key by
   default (`nomic-embed-text-v1.5`, 768d, baked into the image);
   `RASA_EMBEDDINGS=auto` upgrades to OpenAI `text-embedding-3-large`
   (3072d) when a key resolves. Dimensions are per-element: moving an
   element between providers fails `dims_mismatch` until that element's
   corpus is re-ingested. The guard is the feature, not the bug.
4. **Extensions are claims.** Format detection is magic-byte. A `.pdf`
   that is actually HTML will be treated as what it is.

## Soft cross-references (degrade gracefully)

This module stands alone, but plays well with siblings when mounted:

- **Audit ledger** — bulk ingests and deletions are worth recording in
  the project's audit ledger if it has one (`tasks/AUDIT.md` from the
  tasks module, or the project's own `AUDIT.md`); otherwise the kernel's
  document records are the sole record. Convention, not dependency.
- **`rasa.module.jobs`** — a scheduled re-ingest sweep of a corpus
  directory is a natural `job.toml` when the jobs module is mounted;
  without it, run `/ingest` by hand.
