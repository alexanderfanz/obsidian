# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this vault is

This Obsidian vault is a knowledge base about **Treez ecom** — the e-commerce platform built at Treez before it became GapCommerce. The goal is to incrementally build and maintain a structured wiki about this product: its architecture, history, decisions, and domain knowledge.

The **LLM Wiki.md** file in this vault describes the general pattern this wiki follows.

## The pattern (summary)

Three layers:
1. **Raw sources** — immutable source documents the LLM reads but never modifies
2. **Wiki** — LLM-maintained markdown files (summaries, entity pages, concept pages, synthesis). The LLM owns this layer entirely.
3. **Schema** — a config document (this CLAUDE.md, or an AGENTS.md) that defines wiki structure, conventions, and workflows

Two mandatory index files the LLM keeps current:
- **index.md** — content catalog, organized by category, updated on every ingest; used as the first step when answering queries
- **log.md** — append-only chronological record of ingests, queries, and lint passes; each entry prefixed `## [YYYY-MM-DD] <type> | <title>` for easy grep/parsing

Three core operations:
- **Ingest** — read a new source, integrate it into the wiki (may touch 10–15 pages), update index, append to log
- **Query** — read index to find relevant pages, synthesize answer; file good answers back as wiki pages so they compound
- **Lint** — health-check wiki for contradictions, stale claims, orphan pages, missing cross-references, data gaps

## Repo document workflow

Each Treez ecom repo is analyzed independently (with full codebase context) and produces a structured document saved to `raw/<repo-name>.md`. Wiki pages are then built here by ingesting all raw documents together, enabling cross-repo synthesis.

**Template:** `raw/templates/repo-doc-template.md`

### Generating a repo document (do this inside each repo)

1. Open Claude Code in the repo:
   ```bash
   cd /path/to/repo
   claude
   ```
2. Run this prompt:
   > "Read this codebase and generate a repo document following the template at `/Users/alex/obsidian/treez/raw/templates/repo-doc-template.md`. Save the output to `/Users/alex/obsidian/treez/raw/<repo-name>.md`."
3. Optionally guide the focus: *"prioritize the checkout flow and payment logic"* if the repo is large.

For best results, paste the above prompt into each repo's `CLAUDE.md` so it's repeatable without looking it up.

### Building the wiki from raw docs (do this here)

Once one or more raw docs are ready, say **"ingest"** and Claude will read all raw documents together and update the wiki with cross-repo synthesis — shared concepts, service dependencies, data flow, and a page mapping third-party service overlap across repos.

---

## When instantiating a wiki from this pattern

Work with the user to define:
- Directory structure (e.g. `raw/`, `wiki/`, `raw/assets/`)
- Page conventions (frontmatter fields, cross-reference style, summary format)
- Domain-specific entity and concept categories for the index
- Ingest workflow specifics (how involved the user wants to be, batch vs. one-at-a-time)
- Any tooling (qmd for search, Dataview for frontmatter queries, Marp for slides)

Everything in the pattern is optional and modular — match the implementation to the user's actual needs.
