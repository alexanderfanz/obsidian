# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this vault is

This Obsidian vault contains **LLM Wiki.md** — a pattern document describing how to build a personal knowledge base maintained by an LLM. It is an idea file meant to be shared with an LLM agent so that agent can instantiate a concrete wiki implementation tailored to the user's domain and preferences.

The vault itself is not a wiki yet — it's the seed from which one is built.

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

## When instantiating a wiki from this pattern

Work with the user to define:
- Directory structure (e.g. `raw/`, `wiki/`, `raw/assets/`)
- Page conventions (frontmatter fields, cross-reference style, summary format)
- Domain-specific entity and concept categories for the index
- Ingest workflow specifics (how involved the user wants to be, batch vs. one-at-a-time)
- Any tooling (qmd for search, Dataview for frontmatter queries, Marp for slides)

Everything in the pattern is optional and modular — match the implementation to the user's actual needs.
