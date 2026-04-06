---
name: kb
description: "Use when compiling raw sources into the wiki, querying the knowledge base, running health checks or lint on the wiki, evolving or improving wiki coverage, or when user says 'compile', 'update the wiki', 'query', 'lint', 'health check', 'evolve', 'what's missing', or 'suggest improvements'. Also triggers on questions that should be answered from the knowledge base."
---

## Overview

Main operating skill for LLM-maintained knowledge bases. Four workflows: **compile**, **query**, **lint**, **evolve**. All opus-orchestrated with model-aware subagent dispatch.

**First action in every invocation:** read `kb.yaml` from the project root. If missing, tell the user to run `kb-init` and stop.

## Obsidian-Native Formatting

All wiki output MUST use Obsidian-native conventions:

- YAML frontmatter delimited by `---`
- `[[wikilinks]]` for ALL internal links -- NEVER use markdown-style `[text](url)` links for internal references
- `[[wikilinks|display text]]` when the display name differs from the target article
- Tags in frontmatter: `tags: [concept, topic]`
- Aliases in frontmatter: `aliases: [alternate name]`
- Image embeds: `![[image-name.png]]`
- Standard markdown for everything else (headings, lists, bold, code blocks, etc.)

## Model Strategy

Always set the `model` parameter explicitly when dispatching subagents.

| Task | Executor | Verifier |
|---|---|---|
| Index scanning, link checking, file diffing | haiku subagents | none |
| Summarizing individual sources | haiku subagents | opus spot-checks |
| Wiki article writing | sonnet subagents | opus reviews before commit |
| Deep research | sonnet subagents | opus orchestrates + verifies |
| Research synthesis / final reports | opus | none |
| Lint issue detection | sonnet subagents | opus prioritizes + filters |
| Query answering | opus | none |
| Consistency checks | sonnet subagents | opus final judgment |

## Workflow 1: Compile

**Trigger:** User adds files to `raw/` and says "compile" or "update the wiki".

### Incremental Detection

1. Read `wiki/_index.md` (lists every raw source with its last-compiled hash)
2. Scan `raw/` recursively with Glob
3. Diff against index: identify **new**, **changed**, and **deleted** sources
4. Only process what changed -- never recompile the entire `raw/` folder

### Per-File Extraction (sonnet subagents)

For each new or changed source, dispatch a sonnet subagent to:

1. Read the raw source
2. Determine type (article, paper, transcript, dataset, etc.)
3. Extract concepts, claims, entities, and relationships
4. Return a structured extraction object

### Opus Orchestration

After extractions complete, opus:

5. New concepts -> create wiki article
6. Existing concepts -> update and enrich the article with new information
7. Add source backlinks to every article touched
8. Auto-categorize articles into folders based on topic

### Wiki Article Format

```markdown
---
title: "Concept Name"
aliases: [alternate name, abbreviation]
tags: [domain, topic]
sources:
  - "[[raw/articles/source-file.md]]"
created: 2026-04-03
updated: 2026-04-03
---

# Concept Name

Core explanation of the concept.

## Details

Detailed information extracted from sources.

## Relationships

- Related to [[Other Concept]] because...
- Contradicts [[Conflicting Idea]] on the point of...
- Builds on [[Foundation Concept]]

## Sources

- [[raw/articles/source-file.md]] -- key claims extracted from this source
```

### Index Maintenance

After every compile, update these files:

- **`wiki/_index.md`** -- master list: article name, one-line summary, compiled-from hash
- **`wiki/_sources.md`** -- mapping from raw sources to wiki articles they contributed to
- **`wiki/_categories.md`** -- auto-maintained category tree reflecting folder structure

## Workflow 2: Query

**Trigger:** User asks a question about the knowledge base.

### Step 1: Ask Query Depth

Present three options:

- **Quick** -- wiki-only. Read indexes, find relevant articles, synthesize an answer. No external calls.
- **Standard** -- wiki + web. Start from wiki, use WebSearch/WebFetch to fill gaps.
- **Deep** -- full research pipeline. Invoke `/research` for outline, `/research-deep` dispatches sonnet subagents, opus orchestrates and verifies.

### Step 2: Execute at Chosen Depth

Run the appropriate pipeline. For Standard and Deep, clearly mark which information came from the wiki and which came from external sources.

### Step 3: Render Output

Format the answer according to `kb.yaml` output preferences (markdown, Marp, matplotlib, etc.) and save to `output/`.

### Step 4: Smart Suggest Filing

After delivering the answer, evaluate whether the output contains genuinely new knowledge. Only suggest filing back into the wiki when:

- External data was pulled in (Standard/Deep queries)
- New connections between existing concepts were discovered
- Wiki content was contradicted or updated by fresher information

Do NOT suggest filing when the output is just a reformatting or summary of existing wiki content. If the user agrees to file, run Compile on the output.

## Workflow 3: Lint

**Trigger:** "health check", "lint the wiki", or invoked via `/loop` or `/schedule`.

### Checks (sonnet subagents in parallel)

1. **Broken links** -- `[[wikilinks]]` pointing to non-existent articles
2. **Orphan articles** -- wiki articles with zero inbound links
3. **Orphan sources** -- files in `raw/` that were never compiled
4. **Stale articles** -- source file changed since the article was last compiled
5. **Consistency** -- conflicting claims across different articles
6. **Missing backlinks** -- links that should be bidirectional but aren't
7. **Sparse articles** -- articles below ~200 words

### Coverage Density Scoring

For every article, compute a coverage score and tag it `high`, `medium`, or `low`:

| Signal | Weight |
|--------|--------|
| Number of raw sources that contributed to the article | High |
| Word count (high ≥ 400, medium 200-399, low < 200) | Medium |
| Number of inbound `[[wikilinks]]` from other articles | Medium |
| Number of outbound `[[wikilinks]]` to other articles | Low |

**Scoring:**
- `high` -- strong source backing, substantial content, well-linked
- `medium` -- partially covered; one or more signals are weak
- `low` -- thin article: few sources, short content, or isolated from the rest of the wiki

Write the coverage tag into each article's frontmatter during lint:

```yaml
coverage: high | medium | low
```

This tag lets Claude use articles efficiently during Query: high-coverage articles are read with confidence; low-coverage articles are treated as provisional and supplemented with web search when the user picks Standard depth.

After tagging, report a coverage summary:
- Total articles by coverage level
- Low-coverage articles as a prioritized enrichment list

### Opus Collation

Opus collates all findings, filters false positives, and prioritizes by severity (critical > warning > info).

### Output

Save to `output/lint-YYYY-MM-DD.md`. Group issues by severity, include suggested fixes, coverage summary, and ask the user if they want auto-fix applied.

## Workflow 4: Evolve

**Trigger:** "evolve the wiki", "suggest improvements", or "what's missing".

### Process

1. Opus reads `wiki/_index.md`, `wiki/_categories.md`, and samples articles
2. Dispatch sonnet subagents to analyze article clusters for:
   - **Gaps** -- concepts referenced in articles that lack their own article
   - **Connections** -- cross-category relationships not yet explored
   - **Missing data** -- claims that could be verified or enriched via web search
   - **Questions** -- interesting unanswered questions surfaced by the existing content
3. Opus collates, deduplicates, and ranks suggestions by value
4. Present as a numbered list with brief rationale for each
5. User picks items -> Claude executes via Compile, Query, or web search as appropriate

## Common Mistakes

- Running compile on the entire `raw/` folder when only a few files changed -- always use incremental detection
- Using markdown-style links `[text](url)` instead of `[[wikilinks]]` for internal references
- Skipping index updates (`_index.md`, `_sources.md`, `_categories.md`) after compile
- Suggesting to file every query output back into the wiki -- only genuinely new knowledge qualifies
- Defaulting to Deep query depth for simple factual questions -- try Quick first
