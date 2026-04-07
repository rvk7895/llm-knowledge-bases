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
- **`wiki/_evolution.md`** -- append-only log of auto-evolve actions: `date | trigger | action | articles affected`

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

### Step 4: Auto-Evolve (background opus subagent)

After delivering the answer to the user, **immediately dispatch a background opus subagent** (via Agent tool with `run_in_background: true`). Do NOT ask the user for permission — this runs silently.

The subagent receives:
- The original query
- The full response (including sources used and depth level)
- The current `wiki/_index.md`

The subagent decides and acts:

1. **New knowledge?** — Did the response contain information not already in the wiki? (External data from Standard/Deep queries, new connections between concepts, corrections to existing articles, newly surfaced entities or claims)
2. **If no** — exit silently. Log nothing. The query was just a read.
3. **If yes** — act:
   - **New concept** → create a wiki article following the standard article format, place in the right category folder
   - **Enrichment** → update existing articles with new information, add source backlinks
   - **Correction** → update articles where the response contradicted or superseded wiki content
   - **New connections** → add `[[wikilinks]]` between articles that the query revealed are related
   - Update `wiki/_index.md`, `wiki/_sources.md`, `wiki/_categories.md`
   - Append a one-line entry to `wiki/_evolution.md`: `YYYY-MM-DD | query | action taken | articles affected`

**Constraints:**
- Never delete or downgrade existing content — only add or refine
- Never create stub articles under 100 words — if there isn't enough to say, add the knowledge to an existing article instead
- Quick-depth queries rarely produce new knowledge — the subagent should almost always exit silently for these
- The subagent must re-read any article it plans to modify (not rely on cached state)

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

### Opus Collation

Opus collates all findings, filters false positives, and prioritizes by severity (critical > warning > info).

### Output

Save to `output/lint-YYYY-MM-DD.md`. Group issues by severity, include suggested fixes, and ask the user if they want auto-fix applied.

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

## Handling X/Twitter Links

When the user pastes an `x.com` or `twitter.com` URL and wants it added to the knowledge base:

### Step 1: Check if Smaug is available

Run via Bash: `which bird 2>/dev/null && test -f smaug.config.json && echo "smaug-ready" || echo "smaug-missing"`

### Step 2a: Smaug IS available

1. Extract the tweet ID from the URL (the numeric part after `/status/`)
2. Run: `npx smaug fetch <tweet_id>` then `npx smaug process`
3. Smaug outputs markdown with frontmatter to its `knowledge/` directory
4. Copy the output to `raw/articles/` and trigger Compile

### Step 2b: Smaug is NOT available

Tell the user you cannot directly fetch X/Twitter content, then present these options:

1. **Install Smaug** (recommended) — `git clone https://github.com/alexknowshtml/smaug && cd smaug && npm install`, then `npx smaug setup` to configure X session cookies (`auth_token` + `ct0` from browser DevTools → Cookies → x.com). Note: uses session cookies, technically violates X TOS but practical risk for personal read-only use is very low.
2. **Manual paste** — ask the user to copy-paste the tweet/thread text. Save to `raw/articles/x-<tweet_id>.md` with the tweet URL as a source link.
3. **Thread Reader App** — for threads, suggest pasting the URL at threadreaderapp.com, then copy the unrolled result.
4. **X Data Export** — for bulk import of own tweets/bookmarks: X Settings → Download your data (TOS-compliant, 24-48hr wait).

**If the user provides the tweet content directly** (paste, screenshot, or any other method), accept it immediately and proceed — save to `raw/articles/x-<tweet_id>.md` with proper frontmatter and trigger Compile. Do not gatekeep.

### Smaug output format

Smaug produces markdown like:
```markdown
## @username - Title
> Tweet text

- **Tweet:** https://x.com/...
- **Link:** https://...
- **What:** One-line description
```

And knowledge files with YAML frontmatter in `knowledge/articles/` or `knowledge/tools/`. Both formats are valid raw sources for Compile.

## Common Mistakes

- Running compile on the entire `raw/` folder when only a few files changed -- always use incremental detection
- Using markdown-style links `[text](url)` instead of `[[wikilinks]]` for internal references
- Skipping index updates (`_index.md`, `_sources.md`, `_categories.md`) after compile
- Asking the user whether to file query results -- auto-evolve handles this silently in the background
- Defaulting to Deep query depth for simple factual questions -- try Quick first
- Forgetting to dispatch the auto-evolve subagent after a query -- it must always run, even if it usually exits silently
- Refusing to process an X/Twitter link just because Smaug isn't installed -- always offer alternatives and accept manual paste
- Trying to use WebFetch on X/Twitter URLs -- it always fails due to auth walls, don't bother
