---
name: reingest
description: Re-ingest an updated file under an existing doc_id — the kernel bumps the document version and replaces its chunks atomically; this skill stages the file, submits with the doc_id, streams progress, and surfaces old→new version. Triggered when a source file changed — e.g. "/reingest abc123 ./corpus/contract.pdf", "the contract was updated, refresh it", "re-ingest that doc".
---

# /reingest — version-bump an existing document

Same pipeline as `/ingest` — stage → ingest → stream — but submitted
with an existing `doc_id`, so the kernel treats it as an update:
**version bump + atomic chunk replace**, kernel-side. This skill's job
is preserving identity and surfacing the version transition.

Configuration from `.claude/ingest-gate.md`; conventions from
`.claude/ingest-rules.md`.

## Behavior contract

- **Both arguments required** — a `doc_id` and a path. No argument
  guessing: re-ingesting the wrong file under an existing id silently
  replaces a document's content.
- **Verify the record first.** `GET /v1/documents/{doc_id}` before
  ingesting — capture the current name + version, and confirm the id
  exists (`document_not_found` → stop; suggest `/docs` or plain
  `/ingest` if this is actually new content).
- **Excludes still apply.** The gate's `excludes` refuse the path here
  exactly as in `/ingest`.
- **Report the transition**: `<name>: v<old> → v<new>, <chunks> chunks
  (replaced atomically)`. Low-yield warning as in `/ingest`.

## Process

1. Read `.claude/ingest-gate.md`; check the path against `excludes`.
2. `GET {kernel_url}/v1/documents/{doc_id}?element=<element>` → record
   the current version; stop on `document_not_found`.
3. Stage: `POST /v1/files` with the file → `file_id`.
4. `POST /v1/documents` with `{"file_id": …, "doc_id": "<doc_id>",
   "name": "<current name>", "element": "<gate element>"}` (omit
   fields you don't set) → follow the SSE stream as in `/ingest`.
5. Confirm via the record: fetch again, render
   `v<old> → v<new> · <pages> pages · <chunks> chunks`.

## When NOT to use this skill

- **New content** → `/ingest` (a fresh doc_id is correct identity).
- **Content is gone, not changed** → `/doc <id> --delete`.
- **A dims_mismatch re-ingest sweep** (embedding provider changed) →
  that's every document in the element: `/docs` for the list, then
  `/reingest` per document — say up front that it's a sweep.

## What "done" looks like

The same doc_id with a bumped version and fresh chunks, the old→new
transition reported, and no orphaned duplicate record (that's what
plain `/ingest` would have created).
