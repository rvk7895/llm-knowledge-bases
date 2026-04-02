# LLM Knowledge Bases

A Claude Code plugin that turns raw research material into an LLM-maintained Obsidian wiki -- inspired by [Andrej Karpathy's description](https://x.com/karpathy/status/2039805659525644595) of using LLMs as knowledge compilers rather than just code manipulators.

You drop source material into `raw/`, run a single command, and Claude handles the rest: compiling interlinked wiki articles, maintaining indexes and backlinks, answering complex questions at multiple depth levels, and continuously improving the knowledge base over time.

The LLM owns the wiki. You rarely edit it manually -- just explore in Obsidian and keep feeding it raw data.

## How It Works

1. **Ingest** -- Raw documents (articles, papers, repos, YouTube transcripts, images, datasets) go into `raw/`
2. **Compile** -- Claude builds a structured Obsidian vault with summaries, backlinks, concept articles, and auto-generated indexes
3. **Query** -- Three depth levels:
   - **Quick** -- Answers from wiki indexes and summaries alone
   - **Standard** -- Cross-references the full wiki, supplements with web search
   - **Deep** -- Multi-agent research pipeline with parallel web search agents
4. **Output** -- Markdown reports, Marp slides, matplotlib charts -- saved to `output/` and optionally filed back into the wiki
5. **Maintain** -- Automated health checks (broken links, orphans, inconsistencies) and suggestions for new articles and connections

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

| Skill | Purpose |
|-------|---------|
| `kb-init` | One-time setup: scaffolds directories, generates config, writes project files |
| `kb` | Main skill with four workflows: compile, query, lint, evolve |
| `research` | Generate a structured research outline for a topic |
| `research-deep` | Launch parallel agents for deep research on each outline item |
| `research-add-fields` | Add field definitions to an existing research outline |
| `research-add-items` | Add research targets to an existing outline |
| `research-report` | Compile deep research results into a markdown report |

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
