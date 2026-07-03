# `rasa.module.ingest` — content

This is what the module ships. It is the **human workflow** over the
kernel's document-ingest / RAG-query / semantic-memory system (canon
SA-029) — the engine itself lives in the kernel and is deliberately not
wrapped in new plumbing here. The things that vary per project — which
kernel, which element identity, which paths a sweep may touch — are
pulled out into a project-owned `ingest-gate.md`.

## What installs where

| Source | Installs to | Policy | Owner |
|---|---|---|---|
| `content/ingest-rules.md` | `.claude/ingest-rules.md` | file-replace | Element (refreshed on upgrade) |
| `content/skills/{ingest,docs,doc,doc-query,remember,recall,forget,reingest}/` | `.claude/skills/` | directory-mirror | Element |
| `seed/ingest-gate.md.template` | `.claude/ingest-gate.md` | skip-if-exists | **Project** (you fill it) |
| `seed/rasa.lock.json.template` | `.claude/rasa.lock.json` | init-only-with-sha | Project |

There is **no project-owned ledger** seed (contrast the releases
module's `RELEASES.md`): the kernel's document store IS the record, and
`/docs` reads it live.

## The split that makes it portable

- **Element-owned (`content/`)** — the ingest conventions
  (`ingest-rules.md`: when to ingest vs read, documents vs memory,
  namespaces, error codes, caps, the operational gotchas) and the eight
  skills. Identical for every consuming project. Upgrades flow in.
- **Project-owned (`seed/`)** — the `ingest-gate.md`: `kernel_url`,
  `element`, `default_namespace`, `memory_scope`, `ingest_roots`, and
  the load-bearing `excludes` (a sweep must never vectorize secrets).
  The project fills it; never overwritten on upgrade.

## The two halves of "fully work with ingest"

1. **The skills** — the explicit human entry points over the kernel's
   HTTP surface (below).
2. **The prompting discipline** — `ingest-rules.md` tells the model
   that every kernel-driven session already has the injected `kmem`
   tools (`doc_search` / `memory_search` / `memory_store`) and to use
   them unprompted when asked about ingested content. No new tool
   plumbing; the module's job is the workflow + the discipline.

## Skills

- **`/ingest <path|glob|dir>`** — stage → ingest → stream progress,
  per-file isolation, excludes enforced, low-yield warnings.
- **`/docs [element]`** — list document records; failed rows flagged.
- **`/doc <id> [--delete]`** — one record; delete echoes honest
  cascade counts.
- **`/doc-query <prompt>`** — RAG query, grouped citations by default;
  `--context` emits a paste-ready `<DOCUMENT>` block.
- **`/remember <text>`** — store a semantic memory (`--ttl` supported).
- **`/recall <query>`** — similarity-search memories (`--list` to
  enumerate).
- **`/forget <memory_id>`** — delete one memory.
- **`/reingest <doc_id> <path>`** — version-bump re-ingest; surfaces
  old→new version.

## Soft cross-references (siblings, not dependencies)

- **`rasa.module.tasks`** — bulk ingests/deletions can record to
  `tasks/AUDIT.md`; without it, the kernel's records are the record.
- **`rasa.module.jobs`** — a scheduled corpus re-sweep is a natural
  `job.toml`; without it, `/ingest` by hand.

## Mounting into a parent

`rasa.module.ingest` is a `module` (canon Spec §6): a focused capability
mountable into a parent `domain` or `orchestrator`. A parent opts in via
its own `rasa.json`:

```json
"requires": {
  "elements": [
    { "name": "rasa.module.ingest", "version": ">=0.1.0" }
  ]
}
```

The kernel resolver pulls it in dependency order; `bin/init` installs
`content/` + `seed/` per the table above.

## See also

- `content/ingest-rules.md` — the ingest spine (the contract).
- `seed/ingest-gate.md.template` — the per-project adapter.
- Kernel `docs/openapi.yaml` — the wire truth (documents / memories /
  files sections).
- Canon `canon/tasks/…/SA-029-document-vector-memory-service.md` — the
  design lineage.
