---
name: ingest
description: Ingest files into the kernel's document store — stage each file, ingest scoped to the project's element, follow the SSE progress stream, and report per-file results. Accepts a single path, a glob, or a directory sweep; per-file failures stay isolated (one bad file never aborts the sweep). Refuses paths matching the ingest-gate's excludes. Triggered when the user wants content in the corpus — e.g. "/ingest ./corpus", "/ingest contract.pdf", "ingest these files", "add this to the knowledge base".
---

# /ingest — sweep files into the document store

Drive the kernel's ingest pipeline for a human: for each resolved file,
`POST /v1/files` (stage) → `POST /v1/documents` (ingest) → follow the
SSE stream → report `{doc_id, pages, chunks, low_yield_pages}`. The
kernel does the extraction, chunking, and embedding; this skill does the
sweep discipline, the scoping, and the honest reporting.

The authoritative conventions are in `.claude/ingest-rules.md`; the
per-project configuration (kernel URL, element, namespace, sweep roots,
excludes) is in `.claude/ingest-gate.md`. Read both first. If the two
disagree, `ingest-rules.md` wins on conventions, the gate wins on
configuration.

## Behavior contract

- **Read the gate first.** `kernel_url`, `element`, `default_namespace`,
  `ingest_roots`, `excludes` all come from `.claude/ingest-gate.md`.
  Never hardcode `:3001`; never invent an element name. If the gate is
  missing or still the unfilled template on the load-bearing lines
  (element, roots), **stop** and ask the user to fill it.
- **Excludes are absolute.** Any resolved path matching an `excludes`
  glob is refused — even when the user named it explicitly — and the
  refusal is reported with the matching pattern. A sweep must never
  vectorize a `.env`, a key, or `.git` internals.
- **Sweeps stay inside the roots.** A directory or glob argument must
  fall under `ingest_roots`. An explicit single-file path outside the
  roots is allowed (deliberate act), but still checked against
  `excludes`.
- **Per-file isolation.** Each file gets its own stage → ingest →
  stream cycle. A failure (unsupported_mime, ingest_failed, network)
  is recorded for the report and the sweep **continues**. One bad file
  must not abort the batch.
- **Omit absent optional fields.** Build the JSON body with only the
  fields you mean — never `"namespace": ""`. The empty-string bug class
  is real (see ingest-rules.md).
- **Warn, don't fail, on low yield.** `low_yield_pages > 0` means
  scanned pages the kernel couldn't extract (vision fallback needs an
  Anthropic API key kernel-side). Flag it per-file in the report;
  the ingest itself succeeded.
- **Never auto-commit anything.** This skill touches the kernel, not
  the git tree.

## Process

1. **Read** `.claude/ingest-gate.md` + `.claude/ingest-rules.md`.
2. **Resolve the argument** to a file list:
   - Single file → that file.
   - Glob → expand it.
   - Directory → recursive sweep of supported-looking files under it.
   - No argument → **stop** and ask; guessing a sweep root is worse
     than asking.
3. **Filter**: drop (and record) anything matching `excludes`; verify
   sweeps fall under `ingest_roots`. Report what was dropped and why —
   silent filtering reads as "ingested everything."
4. **Per file**, sequentially:
   a. Stage: `curl -sS -X POST {kernel_url}/v1/files -F "file=@<path>"`
      → capture `file_id`.
   b. Ingest: `POST {kernel_url}/v1/documents` with JSON
      `{"file_id": …, "name": "<basename>", "element": "<gate element>",
      "namespace": "<gate default_namespace>"}` (omit `namespace` if the
      gate leaves it default; include `metadata` only if given) →
      capture 202 `{task_id, doc_id, stream_url}`.
   c. Stream: `curl -N {kernel_url}/v1/commands/{task_id}/stream` —
      plain `text/event-stream` with standard
      `started`/`text`/`result`/`end` events. Render progress lines as
      they arrive; the `result` event carries the final counts.
   d. Record `{name, doc_id, pages, chunks, low_yield_pages}` or the
      failure `{name, error_code, message}`.
5. **Report** a per-file table: name · doc_id · pages · chunks ·
   low-yield warnings · status. Then one summary line:
   `N ingested, M failed, K refused (excludes)`. Failures show their
   error code and the kernel's stored message — no retry without the
   user asking.

## Error handling

Per `.claude/ingest-rules.md` → "Error codes." Specifically:
`unsupported_mime` → report the file and move on (magic-byte detection —
the extension was a claim). `ingest_failed` → show the stored error.
`rate_limited` → back off once, then surface. A connection failure to
`kernel_url` → stop the sweep (nothing else will succeed either) and
report which files remain unprocessed.

## When NOT to use this skill

- **Re-ingesting an updated file with a known doc_id** → `/reingest`
  (preserves identity, bumps version).
- **Storing a one-line fact or decision** → `/remember`. If it fits in
  a sentence, it's a memory, not a document.
- **A file you just need to read once** → the Read tool. Ingest is for
  corpora you'll query repeatedly by meaning.

## What "done" looks like

Every resolved file either ingested (doc_id + counts reported), failed
(error code reported), or refused (exclude pattern reported) — and the
totals line matches the file list. Nothing silently skipped. Low-yield
pages flagged wherever vision fallback wasn't available.
