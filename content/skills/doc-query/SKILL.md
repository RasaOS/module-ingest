---
name: doc-query
description: RAG-query the element's document corpus and render cited hits — grouped by document with name · page · distance citations by default; --context switches to a ready-to-paste <DOCUMENT> context block. Wraps POST /v1/documents/query scoped to namespaces:["docs"]. Triggered when the user asks the corpus a question — e.g. "/doc-query what does the MSA say about termination?", "/ask-docs …", "search the corpus for X", "what do the ingested docs say about Y?".
---

# /doc-query — ask the document corpus

Embed-and-search the element's corpus via
`POST {kernel_url}/v1/documents/query` and render the organized hits
with citations. The kernel retrieves; **you answer** — synthesize from
the returned chunks and cite `name · page · distance` for every claim
you take from them.

Configuration (kernel URL, element) comes from `.claude/ingest-gate.md`.
Conventions from `.claude/ingest-rules.md`.

## Behavior contract

- **Default body**: `{"prompt": "<question>", "element": "<gate
  element>", "organize": "grouped", "namespaces": ["docs"]}`. The
  namespace filter is deliberate — an unfiltered query spans ALL
  namespaces and memories would surface next to documents. Widen it
  only when the user asks for everything.
- **Omit what you don't set.** No `""` fields, no `k: 0`, no
  `window: ""` — absent means default (k=8, window +0.25 past the best
  hit). `k` caps at 50.
- **`--context` mode**: send `"organize": "context"` (honor
  `--budget <n>` as `token_budget`, default 4000) and print the
  returned `<DOCUMENT>` block verbatim inside a fenced block for
  pasting into a prompt. Surface the `truncated` flag when set.
- **`--since <ts>`** → `since_ts`. **`--k <n>`** → `k`.
- **Answer from the hits, cite the hits.** If the hits don't support an
  answer, say the corpus doesn't cover it — don't fill the gap from
  general knowledge without labeling the seam.
- **Empty result** is an answer: report it, suggest checking `/docs`
  (is the material even ingested?) or widening the query.

## Process

1. Read `.claude/ingest-gate.md` for `kernel_url` + `element`.
2. `POST /v1/documents/query` with the body above.
3. **Grouped render** (default): per document group — document name,
   then its best chunks with `p.<page> · d=<distance>` per chunk.
   Synthesize the actual answer above the citations.
4. **Context render** (`--context`): the `<DOCUMENT>`/`<METADATA>`
   block verbatim, fenced; note the token budget used and whether
   truncation occurred.
5. On `embedding_provider_unavailable` (only possible when the kernel
   is pinned `RASA_EMBEDDINGS=openai` with no key): report it as a
   kernel configuration issue — auto/local never hit it.

## When NOT to use this skill

- **You're mid-conversation and have the `doc_search` tool** — just use
  the tool; this skill is the explicit human entry point, not a
  required detour.
- **Searching memories** → `/recall`.
- **Listing records** → `/docs`.

## What "done" looks like

An answer synthesized from retrieved chunks, every borrowed claim
carrying a `name · page` citation, and the retrieval parameters
(element, k, namespaces) stated once so the result is reproducible.
