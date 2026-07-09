# CLAUDE.md — `rasa.module.ingest`

> **Who you are (SA-025).** `rasa.module.ingest` — the RasaOS module for document ingest and similarity-recall memory. Substrate: **RasaOS**; role: **module**. On install `bin/init` renders this into `.claude/rasa-identity.md`; `/whoami` composes the full identity with the project's deployment layer.


Per-repo working contract for Claude sessions opened inside this folder.
Extends `~/.claude/CLAUDE.md` and the workspace `~/rAI/rasa-os/CLAUDE.md`
(the `rasa.tenant.rasaos` tenant's contract); does not override them.

## What you are when you're in this folder

You are working on **`rasa.module.ingest`** — a `module`-kind Element:
the human workflow over the kernel's document-ingest / RAG-query /
semantic-memory system (canon SA-029; kernel TASK-233..239). Sixth
module, sibling to `module.tasks` (the adapter-seam origin) and the
four engineering modules (releases, pipelines, tests, jobs).

A `module` (canon Spec §6) is "a focused capability that extends a
domain or orchestrator, mountable into one or more parents." Parents
pull it in via their own `requires.elements[]`.

## The load-bearing ideas

**1. The engine is the kernel's.** Staging, extraction, chunking,
embedding, vector indexes, RAG organization, memory, and the injected
`kmem` agent tools all live kernel-side. If you find yourself about to
add extraction logic, a chunking strategy, an embedding call, or a new
tool wrapper to `content/` — **stop**. This module is skills +
conventions + the gate. The kernel's `docs/openapi.yaml` is the wire
truth; when a skill and the kernel disagree, fix the skill.

**2. The ingest-gate.** The three genuinely per-project concerns —
which kernel (`kernel_url`; never hardcode `:3001`), which element
identity + scopes, and which paths a sweep may touch — live in the
project-owned `.claude/ingest-gate.md` (seeded `skip-if-exists`).
`content/` references the gate; it hardcodes none of them.

**3. Excludes are load-bearing.** `/ingest` refuses paths matching the
gate's `excludes` even when named explicitly. An ingested `.env` is a
leaked `.env`. Never weaken this in a skill edit.

**4. Honest gotchas stay in the doc.** `ingest-rules.md` names the
persistence window (kernel restarts can lose recent vectors until the
AOF fix lands), the empty-string rule (omit absent fields, never send
`""`), the vision-fallback prerequisites, and `dims_mismatch` =
re-ingest. Don't paper over them; remove each only when the kernel fix
actually lands.

## Soft cross-references — keep them soft

`module.tasks` (audit-ledger entries) and `module.jobs` (scheduled
re-sweeps) are conveniences, not requirements. Do **not** turn a soft
reference into a hard `requires.elements[]` dependency.

## Source of truth

- **`~/rAI/rasa-os/canon/`** — authoritative. Spec §6 defines the
  `module` kind; ELEMENT_CONTRACT.md §7 the install policies; SA-029
  the DocumentManager design.
- **Kernel `docs/openapi.yaml`** — the wire truth for every endpoint
  the skills drive.
- **`content/ingest-rules.md`** — the ingest spine.
- **`rasa.json`** — the formal declaration + install manifest.

## Don'ts

- **Don't rebuild kernel plumbing** (extraction, chunking, embeddings,
  tool injection). Skills drive HTTP; that's the whole job.
- **Don't hardcode a kernel URL, element name, or sweep root** in
  `content/`. They go in the ingest-gate.
- **Don't weaken the excludes discipline.**
- **Don't harden a soft reference into a hard dependency.**
- **Don't `bin/init` this Element into itself.** `content/` is the
  source.
- **Don't push from the Cowork sandbox.** Local commit + tag only; the
  user pushes from their machine (workspace rule).

## How a version bump works

- **Patch (0.1.0 → 0.1.1)** — wording fix, template clarification,
  `bin/*` bug fix. No structural change.
- **Minor (0.1.x → 0.2.0)** — new skill, new seed file, a new
  capability (e.g. skills for formats the kernel gains: pptx, HTML,
  image OCR). Parents may adopt; not breaking.
- **Major (0.x.x → 1.0.0)** — first stable lock-down, or a breaking
  change to the gate format / install shape after 1.0. Parents
  REQUIRED to migrate.

Each bump: edit `VERSION` + `rasa.json#version`, write a CHANGELOG
entry, run `bin/check-manifest`, commit + tag. Add a row to
`~/rAI/rasa-os/elements/CHANGELOG.md` (track #2) and update
`~/rAI/rasa-os/elements/REGISTRY.md`.

## What success looks like

From a consumer project with the module installed and the gate filled:
`/ingest ./corpus` sweeps and reports; `/docs` lists; `/doc-query`
answers with page citations; `/remember` + `/recall` round-trip; and
the model in that project's sessions reaches for `doc_search`
unprompted when asked about ingested content — the rules doc earning
its keep.
