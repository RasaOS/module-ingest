# CHANGELOG — `rasa.module.ingest`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.2 — 2026-07-09

### Added generic `/sync` + `/promote` + `/kit`-aware `bin/init` (canon SA-024)

- `bin/init` now clones the Element source into `<project>/kit/<element>/`; `/sync` smart-pulls upstream, `/promote` smart-pushes local edits back upstream (both directory-mirror → installed into consumers).

## 0.1.1 — 2026-07-09

### `parent_kind` → `[domain, tenant]` (canon SA-023)

- The `orchestrator` kind was folded into `tenant`; this module now mounts into a tenant or a domain (`requires.parent_kind: ["domain", "tenant"]`, was `["domain", "orchestrator"]`).

## 0.1.0 — 2026-07-02 — INITIAL

**The ingest module — the human workflow over the kernel's document /
RAG / semantic-memory system (canon SA-029, kernel TASK-233..239,
kernel branch `feat/document-track` @ v0.30.0-rc).** Sixth `module`-kind
Element, following the `module-releases` shape and the `module-tasks`
adapter-seam pattern.

### What it is

- **Kind:** `module` (canon Spec §6) — opt-in, mountable into a parent
  `domain` or `orchestrator` via the parent's `requires.elements[]`.
  `requires.parent_kind: [domain, orchestrator]`.
- **Contract:** Element Contract v1.3.0. `bin/check-manifest` GREEN;
  conforms to `RasaOS/schema` v0.1.0.

### The boundary decision (load-bearing)

The kernel owns the whole engine — `/v1/files` staging, `/v1/documents`
ingest + CRUD + query, `/v1/memories`, per-element vector indexes,
embedding selection (zero-key local `nomic-embed-text-v1.5` default,
auto-upgrade to OpenAI), and the `kmem` agent tools (`doc_search` /
`memory_search` / `memory_store`) auto-injected into every claude turn.
**This module wraps none of that in new plumbing.** It ships the human
workflow (skills), the prompting discipline (`ingest-rules.md` tells
the model to use the injected tools unprompted), and the conventions.

### Ships

- **Skills ×8** — `/ingest` (stage → ingest → SSE stream, per-file
  isolation, excludes enforced, low-yield warnings), `/docs` (list;
  failed rows flagged with stored error), `/doc` (show / `--delete`
  with honest cascade counts), `/doc-query` (RAG, `organize:"grouped"`
  + `namespaces:["docs"]` default, `--context` for a paste-ready
  `<DOCUMENT>` block), `/remember` (`--ttl`), `/recall` (`--list`),
  `/forget`, `/reingest` (version-bump, old→new surfaced).
- **`content/ingest-rules.md`** — the spine: ingest vs read-directly,
  documents vs memory ("memory = decisions/facts ≤16KB; documents =
  corpora"), the element/namespace scoping model (unfiltered queries
  span ALL namespaces by design), error-code table, caps, and the
  known gotchas carried honestly: the persistence window (`appendonly
  no` — re-ingest after kernel restarts if counts look wrong), the
  empty-string bug class (OMIT absent optional fields, never send
  `""`), vision-fallback prerequisites (ANTHROPIC_API_KEY, API key
  not OAuth), and `dims_mismatch` = re-ingest (the guard, not the bug).
- **`seed/ingest-gate.md.template`** — the project-owned adapter:
  `kernel_url` (never hardcode `:3001`), `element`,
  `default_namespace`, `memory_scope`, `ingest_roots`, and the
  **load-bearing `excludes`** — `/ingest` refuses matching paths even
  when named explicitly; a sweep must never vectorize a `.env`.

### Install shape

- **Element-owned (`element.files[]`, refreshed on upgrade):**
  `ingest-rules.md`, `skills/` ×8.
- **Project-owned (`seed.files[]`):** `ingest-gate.md`
  (`skip-if-exists`) + the stamped `rasa.lock.json`.
- **No project-owned ledger** (contrast releases' `RELEASES.md`): the
  kernel's document store IS the record; `/docs` reads it live.

### Soft cross-references (siblings, not dependencies)

- `module.tasks` → bulk ingests/deletions may record to `tasks/AUDIT.md`.
- `module.jobs` → a scheduled corpus re-sweep as a `job.toml`.

All present-if-mounted, graceful-if-absent. No hard `requires.elements[]`.

### Provenance / decisions

- Built from the 2026-07-02 kernel-session handoff ("building
  `rasa.module.ingest`"); wire truth cross-checked against kernel
  `docs/openapi.yaml` on `feat/document-track` (documents / memories /
  files sections).
- `bin/init` + `bin/check-manifest` carried from `module-releases`
  (init differs only in the module-specific comment header;
  check-manifest byte-identical — the module-family convention).
- **Boundary decision:** agent-tool plumbing stays out — the kernel
  already injects `kmem`; wrapping it would duplicate a live surface.
  Not-yet-ported formats (pptx, HTML extraction, image OCR) are kernel
  follow-ups, not module work.
