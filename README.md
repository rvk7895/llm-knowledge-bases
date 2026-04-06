# LLM Knowledge Bases

A Claude Code plugin that turns raw research material into an LLM-maintained Obsidian wiki -- inspired by [Andrej Karpathy's description](https://x.com/karpathy/status/2039805659525644595) of using LLMs as knowledge compilers rather than just code manipulators.

Two wiki types, one plugin:

- **Research wikis** (`kb`) -- Drop external sources (articles, papers, repos, transcripts) into `raw/` and Claude compiles a structured knowledge base
- **Personal wikis** (`wiki`) -- Drop your own data (journals, notes, messages, Day One exports, Apple Notes) and Claude compiles a map of your mind

The LLM owns the wiki. You rarely edit it manually -- just explore in Obsidian and keep feeding it raw data.

## How It Works

### Research Wiki (`kb`)

1. **Ingest** -- Raw documents (articles, papers, repos, YouTube transcripts, images, datasets) go into `raw/`
2. **Compile** -- Claude builds a structured Obsidian vault with summaries, backlinks, concept articles, and auto-generated indexes
3. **Query** -- Three depth levels: Quick (wiki-only), Standard (wiki + web), Deep (multi-agent research pipeline)
4. **Output** -- Markdown reports, Marp slides, matplotlib charts -- saved to `output/` and optionally filed back into the wiki
5. **Maintain** -- Automated health checks (broken links, orphans, inconsistencies) and suggestions for new articles and connections

### Personal Wiki (`wiki`)

1. **Ingest** -- Personal data (journals, notes, messages) gets parsed into raw entries by an auto-generated Python script
2. **Absorb** -- Claude reads entries as a writer, understanding what they mean and weaving them into wiki articles
3. **Query** -- Ask questions about your life, projects, patterns, and people
4. **Cleanup** -- Parallel subagents audit and enrich every article, restructure diary-driven pages into narrative ones
5. **Breakdown** -- Find and create missing articles for every named person, place, project, and theme

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI), installed and authenticated
- [Obsidian](https://obsidian.md)
- Recommended Obsidian plugins: Web Clipper, Marp Slides, Dataview

## Installation

In Claude Code, run:

```
/plugin marketplace add rvk7895/llm-knowledge-bases
/plugin install kb@llm-knowledge-bases
```

All skills are installed as a single plugin.

## Quick Start

```bash
# Initialize a new knowledge base
/kb-init

# Add sources to raw/, then compile the wiki
/kb compile

# Query the knowledge base
/kb query

# Run a health check
/kb lint

# Deep multi-agent research on a topic
/research <topic>
/research-deep
```

## Available Skills

### Research Wiki

| Skill | Purpose |
|-------|---------|
| `kb-init` | One-time setup: scaffolds directories, generates config, writes project files |
| `kb` | Four workflows: compile, query, lint (with coverage scoring), evolve |
| `kb-connect` | Discover hidden connections, contradictions, and synthesis opportunities across articles |
| `kb-diagram` | Generate Mermaid concept maps, dependency graphs, and timelines from wiki content |
| `kb-learn` | Feynman technique learning: quiz, deep dive, gap analysis, and scan modes |
| `kb-project` | Compile a codebase into a wiki — architecture, decisions, evolution, ownership — reads repo and git history directly |
| `research` | Generate a structured research outline for a topic |
| `research-deep` | Launch parallel agents for deep research on each outline item |
| `research-add-fields` | Add field definitions to an existing research outline |
| `research-add-items` | Add research targets to an existing outline |
| `research-report` | Compile deep research results into a markdown report |

### Personal Wiki

| Skill | Purpose |
|-------|---------|
| `wiki` | Compile personal data (journals, notes, messages, Day One, Apple Notes) into a personal knowledge wiki. Ingest any format, absorb entries into articles, query, cleanup, breakdown. |
| `wiki-reflect` | Synthesize your wiki over time — monthly and yearly reviews, topic evolution, decision outcomes, recurring patterns. |

## Directory Structure (after `/kb-init`)

```
raw/           -- Raw source documents
wiki/          -- LLM-compiled Obsidian vault
output/        -- Query results, slides, charts, reports
kb.yaml        -- Configuration
CLAUDE.md      -- Project instructions for Claude
```

## Attribution

- [Andrej Karpathy](https://x.com/karpathy/status/2039805659525644595) -- Original vision for LLM-maintained knowledge bases
- [Weizhena](https://github.com/Weizhena/Deep-Research-skills) -- Deep Research skills adapted for the research pipeline
