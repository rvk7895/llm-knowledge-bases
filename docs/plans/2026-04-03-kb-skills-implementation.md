# LLM Knowledge Base Skills Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create two Claude Code skills (`kb-init` and `kb`) that enable LLM-maintained Obsidian wikis from raw source documents.

**Architecture:** Two skills as SKILL.md files in `skills/`. Research skills copied from `~/.claude/skills/research*` with attribution. README and CLAUDE.md at project root for repo documentation and future-session context.

**Tech Stack:** Claude Code skills (markdown), YAML config, Obsidian-compatible markdown formatting.

---

### Task 1: Create `kb-init` Skill

**Files:**
- Create: `skills/kb-init/SKILL.md`

**Step 1: Write the skill file**

```markdown
---
name: kb-init
description: Use when setting up a new knowledge base project, bootstrapping an Obsidian vault for LLM-maintained wikis, or when the user says "init kb" or "new knowledge base"
---

# Knowledge Base Init

## Overview

One-time (or re-runnable) setup that bootstraps a knowledge base project as an Obsidian vault. Walks the user through configuration, scaffolds directories, and prepares the project for the `kb` skill workflows.

## Prerequisites

Inform the user:
- Obsidian must be installed (https://obsidian.md)
- Recommended Obsidian plugins: Web Clipper, Marp Slides, Dataview

## Flow

### Step 1: Vault Setup

Ask the user:
- **"Create a new Obsidian vault, or use an existing directory?"**
  - New vault: scaffold `.obsidian/` config directory
  - Existing: verify the directory exists, check for conflicts

### Step 2: Source Gathering Strategy

Ask: **"How do you want to get raw data into the knowledge base?"**

- **Self-serve** -- You drop files into `raw/` yourself. I'll provide guidance per source type:
  - **Web articles**: Install Obsidian Web Clipper browser extension. Clip articles to `raw/articles/`
  - **Academic papers (PDF)**: Download to `raw/papers/`
  - **GitHub repos**: Clone or download snapshots to `raw/repos/`
  - **Local markdown/text**: Copy to `raw/notes/`
  - **Images/diagrams**: Place in `raw/images/`
  - **YouTube transcripts**: Use a transcript tool, save to `raw/transcripts/`
  - **Datasets (CSV, JSON)**: Place in `raw/datasets/`
- **Assisted** -- Tell me what to fetch. I'll use WebFetch for articles, Bash for cloning repos, download PDFs, and pull transcripts.
- **Mixed** -- Some of each.

### Step 3: Output Format Preferences

Ask: **"Which output formats do you want for query results?"**

Present options:
- Markdown articles (default, always on)
- Marp slides (presentation decks)
- Matplotlib/chart images
- HTML pages
- CSV/table exports
- Excalidraw diagrams
- Other (user specifies)

The skill teaches Claude an extensible pattern: for any format, generate the content, save to `output/`, and it's viewable in Obsidian or externally.

### Step 4: Maintenance Cadence

Inform the user about health check options:
- **Daily/hourly**: Use `/loop` (e.g., `/loop 1d kb lint`)
- **Weekly/monthly**: Use `/schedule` to create a cron-based remote agent
- **Manual**: Just ask "lint the wiki" anytime

Ask which they prefer, or skip for now.

### Step 5: Generate Config

Write `kb.yaml` at project root:

```yaml
name: "<user-provided or 'My Knowledge Base'>"
paths:
  raw: raw/
  wiki: wiki/
  output: output/
output_formats:
  - markdown
  - <user selections>
obsidian:
  wikilinks: true
  recommended_plugins:
    - obsidian-web-clipper
    - obsidian-marp-slides
    - dataview
```

### Step 6: Scaffold Directories

Create the directory structure:
```
raw/
  articles/
  papers/
  repos/
  notes/
  images/
  transcripts/
  datasets/
wiki/
output/
```

If new vault, also create `.obsidian/` with minimal config.

### Step 7: Write Project Files

**Write `CLAUDE.md`** at project root with content:
- This is an LLM Knowledge Base project
- Use the `kb` skill for all wiki operations
- Read `kb.yaml` for configuration
- Always use Obsidian-compatible formatting (wikilinks, YAML frontmatter, embeds)
- Never manually edit wiki articles -- use the kb skill workflows

**Write `README.md`** at project root with:
- Project description and purpose
- Prerequisites (Obsidian, recommended plugins)
- Setup instructions (run kb-init)
- Available workflows (compile, query, lint, evolve)
- Directory structure explanation
- Attribution for research skills

### Step 8: Next Steps Guidance

After scaffolding, tell the user:

**"Your knowledge base is ready! Here's what to do next:"**

1. **Add raw sources** -- Drop files into the appropriate `raw/` subdirectory, use Web Clipper, or ask me to fetch content for you
2. **Compile the wiki** -- Once you have sources in `raw/`, say "compile the wiki" (uses the `kb` skill)
3. **Query** -- Ask questions against your wiki at three depth levels (quick, standard, deep)
4. **Lint** -- Run health checks to find broken links, orphans, inconsistencies
5. **Evolve** -- Ask me to suggest improvements, find gaps, and discover connections

If you set up a maintenance cadence, confirm it's active.

## Common Mistakes

- Forgetting to install Obsidian plugins before trying to view Marp slides or Dataview queries
- Putting raw files directly in `wiki/` instead of `raw/` -- the compile workflow handles the conversion
- Editing wiki articles manually instead of letting the compile workflow maintain them
```

**Step 2: Commit**

```bash
git add skills/kb-init/SKILL.md
git commit -m "feat: add kb-init skill for knowledge base bootstrapping"
```

---

### Task 2: Create `kb` Skill

**Files:**
- Create: `skills/kb/SKILL.md`

**Step 1: Write the skill file**

```markdown
---
name: kb
description: Use when operating on an LLM knowledge base -- compiling raw sources into wiki articles, querying the wiki, running health checks, or evolving the wiki with new connections and content
---

# Knowledge Base

## Overview

The main operating skill for LLM-maintained knowledge bases. Four workflows: compile, query, lint, evolve. All opus-orchestrated with model-aware subagent dispatch.

**First:** Read `kb.yaml` at project root for paths and preferences. If it doesn't exist, inform user to run the `kb-init` skill first.

## Obsidian-Native Formatting

All wiki output MUST use:
- YAML frontmatter delimited by `---`
- `[[wikilinks]]` for all internal links (NEVER markdown-style `[text](path)` links)
- `[[wikilinks|display text]]` when display name differs from article title
- Tags in frontmatter: `tags: [concept, topic]`
- Aliases in frontmatter: `aliases: [alternate name, abbreviation]`
- Image embeds: `![[image-name.png]]`
- Standard markdown for headings, lists, code blocks, tables

## Model Strategy

Opus orchestrates every workflow. Delegate volume work to cheaper models. Always verify before writing to wiki.

| Task | Executor | Verifier |
|------|----------|----------|
| Index scanning, link checking, file diffing | **haiku** subagents | -- (mechanical) |
| Summarizing individual sources | **haiku** subagents | **opus** spot-checks |
| Wiki article writing | **sonnet** subagents | **opus** reviews before committing |
| Deep research | **sonnet** subagents | **opus** orchestrates + verifies |
| Research synthesis / final reports | **opus** | -- |
| Lint issue detection | **sonnet** subagents | **opus** prioritizes + filters |
| Query answering | **opus** | -- |
| Consistency checks | **sonnet** subagents | **opus** makes final judgment |

When dispatching subagents, always set the `model` parameter explicitly.

## Workflow 1: Compile

**Trigger:** User adds new files to `raw/` and says "compile," "update the wiki," or similar.

### Incremental Detection

1. Read `wiki/_index.md` -- it lists every raw source with its last-compiled hash
2. Scan `raw/` directory recursively with Glob
3. Diff against the index: identify new, changed, and deleted files
4. Only process what's changed

### Per-File Compilation

For each new/changed file, dispatch a **sonnet** subagent to:

1. Read the raw source file
2. Determine its type (article, paper, repo, dataset, image, transcript, etc.)
3. Extract key concepts, claims, entities, relationships
4. Return structured extraction as JSON

Then **opus** orchestrator:

5. For each concept that doesn't have a wiki article -- create one
6. For each concept that already exists -- update/enrich with new information from this source
7. Add source backlinks
8. Auto-categorize into folder structure (evolve categories organically as wiki grows)

### Wiki Article Format

```
---
title: <Article Title>
aliases: [<alternate names>]
tags: [<relevant tags>]
sources: [<raw file paths that contributed>]
last_updated: <YYYY-MM-DD>
---

# <Article Title>

<Article body with [[wikilinks]] to related concepts>

## Sources
- [[Sources/<source-name>]] - <brief description>
```

### Index Maintenance

After compilation, update these files:
- `wiki/_index.md` -- master list: every article with a one-line summary and its compiled-from hash
- `wiki/_sources.md` -- every raw source and which wiki articles it contributed to
- `wiki/_categories.md` -- auto-maintained category tree reflecting current folder structure

These indexes are critical -- they're what makes Q&A work without RAG. Claude reads indexes first to find relevant articles.

## Workflow 2: Query

**Trigger:** User asks a question against the wiki.

### Step 1: Ask Query Depth

- **Quick** -- Wiki-only. Read `wiki/_index.md`, find relevant articles, read them, synthesize answer. Fast, no external calls.
- **Standard** -- Wiki + web search. Read wiki first, then supplement gaps with WebSearch/WebFetch.
- **Deep** -- Full research pipeline. Invoke the `/research` skill to generate an outline, then `/research-deep` dispatches **sonnet** subagents for structured research. **Opus** orchestrates and verifies all findings. Best for big open-ended questions.

### Step 2: Execute

At the chosen depth. For Deep queries, the research skills handle orchestration -- see `skills/research/` for details.

### Step 3: Render Output

Read `output_formats` from `kb.yaml`. Render in the user's preferred format:
- **Markdown**: Write `.md` to `output/`
- **Marp slides**: Write Marp-formatted `.md` with `marp: true` frontmatter to `output/`
- **Matplotlib**: Generate a Python script, run it, save image to `output/`
- **Other formats**: Follow the same pattern -- generate content, save to `output/`

For any format not in the list, ask the user how they'd like it rendered and follow the pattern.

### Step 4: Smart Suggest Filing

If the output contains genuinely new knowledge (not just a reformatting of existing wiki articles), suggest:

**"This adds new insight about X. Want me to file it into the wiki?"**

Only suggest when:
- Standard/Deep queries pulled in external data not in the wiki
- The synthesis revealed a new connection between existing concepts
- The answer contradicts or significantly updates existing wiki content

If the user agrees, run the Compile workflow on the new material.

## Workflow 3: Lint

**Trigger:** User says "health check," "lint the wiki," or it runs via `/loop`/`/schedule`.

### Checks

Dispatch **sonnet** subagents in parallel, each handling a category of checks:

1. **Broken links** -- Find `[[wikilinks]]` pointing to non-existent articles
2. **Orphan articles** -- Articles with zero inbound `[[wikilinks]]` from other articles
3. **Orphan sources** -- Files in `raw/` that never got compiled into the wiki
4. **Stale articles** -- Source material changed since last compile (hash mismatch)
5. **Consistency** -- Conflicting claims across articles (e.g., different dates for the same event)
6. **Missing backlinks** -- References that should be bidirectional but aren't
7. **Sparse articles** -- Articles below ~200 words that could be enriched

**Opus** collates results, filters false positives, prioritizes by severity.

### Output

Write report to `output/lint-YYYY-MM-DD.md`:
- Issues grouped by severity (critical, warning, suggestion)
- For each issue: what's wrong, where, and a specific suggested fix
- Ask user: "Want me to auto-fix any of these?"

## Workflow 4: Evolve

**Trigger:** User says "evolve the wiki," "suggest improvements," or "what's missing."

### Flow

1. **Opus** reads `wiki/_index.md`, `wiki/_categories.md` to understand current state
2. Dispatch **sonnet** subagents to analyze clusters of articles, looking for:
   - **Gaps** -- Concepts referenced in `[[wikilinks]]` but never given their own article
   - **Connections** -- Articles in different categories that share unexplored relationships
   - **Missing data** -- Claims or fields that could be filled/verified with a web search
   - **Questions** -- Interesting questions the wiki could answer but doesn't yet
3. **Opus** collates suggestions, deduplicates, ranks by value
4. Present ranked suggestions to user as a numbered list
5. User picks which to pursue
6. Claude executes: create new articles (Compile), answer questions (Query), fill gaps (web search + Compile)

## Common Mistakes

- Running compile on the entire `raw/` directory when only a few files changed -- always check the index first
- Writing markdown-style links instead of `[[wikilinks]]` -- Obsidian won't resolve them
- Skipping index updates after compile -- this breaks Q&A discovery
- Filing every query output back into the wiki -- only file genuinely new knowledge
- Running Deep queries for simple factual questions -- use Quick first, escalate if needed
```

**Step 2: Commit**

```bash
git add skills/kb/SKILL.md
git commit -m "feat: add kb skill with compile, query, lint, evolve workflows"
```

---

### Task 3: Copy Research Skills with Attribution

**Files:**
- Create: `skills/research/SKILL.md` (copied from `~/.claude/skills/research/SKILL.md`)
- Create: `skills/research/validate_json.py` (copied from `~/.claude/skills/research/validate_json.py`)
- Create: `skills/research-deep/SKILL.md` (copied from `~/.claude/skills/research-deep/SKILL.md`)
- Create: `skills/research-add-fields/SKILL.md` (copied from `~/.claude/skills/research-add-fields/SKILL.md`)
- Create: `skills/research-add-items/SKILL.md` (copied from `~/.claude/skills/research-add-items/SKILL.md`)
- Create: `skills/research-report/SKILL.md` (copied from `~/.claude/skills/research-report/SKILL.md`)

**Step 1: Copy all research skill files**

```bash
cp -r ~/.claude/skills/research skills/research
cp -r ~/.claude/skills/research-deep skills/research-deep
cp -r ~/.claude/skills/research-add-fields skills/research-add-fields
cp -r ~/.claude/skills/research-add-items skills/research-add-items
cp -r ~/.claude/skills/research-report skills/research-report
```

**Step 2: Add attribution header to each copied SKILL.md**

Add to the top of each file (after frontmatter):

```markdown
> **Attribution:** This skill was originally authored by [ORIGINAL AUTHOR - ask user].
> Included in this repo with permission for use in the Deep query workflow.
```

**Step 3: Ask user for original author details**

Before committing, ask the user who authored the research skills so attribution is correct.

**Step 4: Commit**

```bash
git add skills/research/ skills/research-deep/ skills/research-add-fields/ skills/research-add-items/ skills/research-report/
git commit -m "feat: add research skills for deep query workflow (with attribution)"
```

---

### Task 4: Write README.md

**Files:**
- Create: `README.md`

**Step 1: Write README**

Content should cover:
- Project title and description (LLM-powered personal knowledge bases as Obsidian wikis)
- Prerequisites (Obsidian, recommended plugins, Claude Code with skills)
- Quick start (run `kb-init`, add sources, compile)
- Available workflows with brief descriptions (compile, query, lint, evolve)
- Directory structure explanation
- Configuration (`kb.yaml` reference)
- Research skills attribution
- The concept: raw data → LLM compilation → wiki → Q&A → evolution loop

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add project README"
```

---

### Task 5: Write CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

**Step 1: Write project-level Claude instructions**

Content:
- This is an LLM Knowledge Base project
- Always read `kb.yaml` for configuration before operating
- Use `kb-init` skill for setup, `kb` skill for all wiki operations
- Always use Obsidian-compatible formatting (wikilinks, YAML frontmatter, image embeds)
- Never manually edit wiki articles outside of the kb skill workflows
- Follow the model strategy: opus orchestrates, haiku/sonnet for subagent work
- Research skills in `skills/research*` power the Deep query workflow

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md with project-level instructions"
```

---

### Task 6: Manual Verification

**Step 1: Verify directory structure**

```bash
find . -type f | head -30
```

Expected:
```
./kb.yaml          # (not yet -- created by kb-init at runtime)
./README.md
./CLAUDE.md
./skills/kb-init/SKILL.md
./skills/kb/SKILL.md
./skills/research/SKILL.md
./skills/research/validate_json.py
./skills/research-deep/SKILL.md
./skills/research-add-fields/SKILL.md
./skills/research-add-items/SKILL.md
./skills/research-report/SKILL.md
./docs/plans/2026-04-03-kb-skills-design.md
./docs/plans/2026-04-03-kb-skills-implementation.md
```

**Step 2: Verify each SKILL.md has valid frontmatter**

```bash
for f in skills/*/SKILL.md; do echo "--- $f ---"; head -4 "$f"; echo; done
```

Each should have `---`, `name:`, `description:`, `---`.

**Step 3: Verify research skills have attribution**

```bash
grep -l "Attribution" skills/research*/SKILL.md
```

Should match all 5 research skill files.
