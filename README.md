# RasaOS Module · Ingest

**Canonical name:** `rasa.module.ingest`
**Repo / folder:** `module-ingest`
**Kind:** `module` (canon Spec §6 — *a focused capability that extends a domain or orchestrator, mountable into one or more parents*)
**Contract:** Element Contract v1.3.0
**Version:** 0.1.0
**Status:** Live. The human workflow over the kernel's document / RAG / memory system (canon SA-029).

## What this is

A portable human workflow over the kernel's **DocumentManager** — the
ingest, RAG-query, and semantic-memory system shipped on the kernel's
document track (canon SA-029, kernel TASK-233..239). The way
`rasa.module.tasks` wraps the task lifecycle, this module wraps the
ingest lifecycle:

```
/ingest ./corpus  →  kernel: stage → extract → chunk → embed → index
/docs, /doc       →  the live document records
/doc-query "…"    →  cited answers (name · page · distance)
/remember /recall /forget  →  semantic memory
/reingest         →  version-bump updates
```

## The engine is the kernel's

Everything heavy already exists kernel-side: `POST /v1/files` staging,
`POST /v1/documents` ingest with magic-byte format detection (pdf, docx,
xlsx, csv, json/jsonl, text, markdown), per-element vector indexes with
zero-key local embeddings by default, `POST /v1/documents/query` RAG
with organized cited output, `/v1/memories` semantic memory, and the
`kmem` agent tools (`doc_search` / `memory_search` / `memory_store`)
auto-injected into every claude turn.

This module ships **no new plumbing**. It ships:

1. **Eight skills** — the explicit human entry points over that HTTP
   surface.
2. **`ingest-rules.md`** — the prompting spine: when to ingest vs read
   directly, documents vs memory, namespace conventions, error codes,
   caps, operational gotchas — and the standing instruction to use the
   injected tools unprompted when asked about ingested content.
3. **The ingest-gate** — the per-project adapter seam.

## The ingest-gate — how it adapts per project

`.claude/ingest-gate.md` (seeded `skip-if-exists`, project-owned)
declares the three things that genuinely vary:

- **Which kernel** — `kernel_url` (+ auth source, if any).
- **Which element** — the identity every document, memory, and query is
  scoped to, plus `default_namespace` and `memory_scope`.
- **What a sweep may touch** — `ingest_roots`, and the **load-bearing
  `excludes`**: `/ingest` refuses paths matching them (`.env*`, keys,
  `node_modules`, `.git`, …) even when named explicitly. A sweep must
  never vectorize a secret.

## Plays well with its siblings (soft references)

- **`rasa.module.tasks`** — bulk ingests/deletions can record to the
  audit ledger; freeform works without it.
- **`rasa.module.jobs`** — a scheduled corpus re-sweep is a natural job.

Every cross-reference degrades gracefully — no hard dependency.

## Install / mount

A parent domain or orchestrator opts in via its own `rasa.json`:

```json
"requires": {
  "elements": [
    { "name": "rasa.module.ingest", "version": ">=0.1.0" }
  ]
}
```

The kernel resolver pulls it in dependency order; the declarative
install applies `element.files[]` + `seed.files[]`. For local install
testing, `bin/init <target-dir>` copies the content per the manifest.
See [`content/README.md`](content/README.md) for the full file-by-file
map.

## Layout

- `content/ingest-rules.md` — the ingest spine (Element-owned).
- `content/skills/` — `/ingest`, `/docs`, `/doc`, `/doc-query`,
  `/remember`, `/recall`, `/forget`, `/reingest`.
- `seed/ingest-gate.md.template` — the per-project adapter.
- `bin/init`, `bin/check-manifest` — the canonical installer + manifest
  checker.

## See also

- Kernel `docs/openapi.yaml` — the wire truth (documents / memories / files).
- Canon SA-029 — the DocumentManager design lineage.
- `~/rAI/rasa-os/elements/module-tasks/` — the first module; the adapter-seam pattern's origin.
- `~/rAI/rasa-os/elements/module-releases/` — the module shape this follows.
- `~/rAI/rasa-os/elements/REGISTRY.md` — live Element registry.
