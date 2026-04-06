---
name: kb-diagram
description: "Use when the user wants to visualize their knowledge base, generate concept maps, architecture diagrams, timelines, or dependency graphs from wiki content. Triggers on 'diagram', 'visualize', 'map the wiki', 'concept map', 'show relationships', 'draw a timeline', or 'graph this'."
---

## Overview

Reads wiki articles and generates Mermaid diagrams that visualize structure, relationships, and flow. Output is saved to `output/` as Obsidian-compatible markdown files with embedded Mermaid syntax — Obsidian renders these natively with the Mermaid plugin.

**First action:** read `kb.yaml` for paths. If missing, tell the user to run `kb-init` and stop.

## Diagram Types

Ask the user which type they want, or infer from context:

| Type | Best for | Mermaid syntax |
|------|----------|----------------|
| **Concept map** | How ideas relate across the whole wiki | `graph TD` |
| **Dependency graph** | What concepts depend on / build from what | `graph LR` |
| **Timeline** | Events, versions, or milestones in sequence | `timeline` |
| **Flowchart** | A process or workflow described in an article | `flowchart TD` |
| **Entity relationship** | Structured data (people, orgs, systems) | `erDiagram` |
| **Sequence diagram** | Step-by-step interactions | `sequenceDiagram` |

If the user says "diagram the wiki" without specifying, default to **concept map** of the whole wiki.

## Model Strategy

| Task | Executor |
|------|----------|
| Extract entities and relationships from articles | sonnet subagents |
| Generate Mermaid syntax | sonnet |
| Review for accuracy and clarity | opus |

## Workflow

### Step 1: Determine Scope

Ask or infer:
- **Whole wiki** -- diagram all articles and their relationships (concept map)
- **Single article** -- diagram the internal structure or process described in one article
- **Category** -- diagram all articles in a folder/category
- **Topic** -- diagram articles matching a user-specified theme

### Step 2: Extract Structure (sonnet subagents)

For the chosen scope, dispatch **sonnet** subagents to read each article and extract:

- Article name (node)
- Existing `[[wikilinks]]` to other articles (edges)
- Key entities mentioned (sub-nodes for entity diagrams)
- Sequential steps if the article describes a process (for flowcharts)
- Dates or versions if the article describes events (for timelines)

Return structured extraction as JSON.

### Step 3: Generate Diagram (sonnet)

Sonnet assembles the Mermaid diagram from the extracted structure.

**Concept map rules:**
- Each article becomes a node: `ArticleName["Article Name"]`
- Each `[[wikilink]]` becomes a directed edge: `ArticleA --> ArticleB`
- Cluster articles in the same folder using `subgraph`
- Keep node labels short -- use the article's title, not its full path
- If the wiki has more than 30 articles, ask the user if they want the full graph or a filtered view

**General rules:**
- Always add a title comment at the top: `%% Concept Map — YYYY-MM-DD`
- Use meaningful edge labels when the relationship type is known: `-->|builds on|`, `-->|contradicts|`, `-->|see also|`
- Prefer left-to-right layout (`graph LR`) for dependency graphs, top-down (`graph TD`) for hierarchies

### Step 4: Opus Review

Opus reviews the generated diagram for:
- Accuracy (edges match actual wiki links)
- Clarity (no orphan nodes, no unreadable label collisions)
- Completeness (major articles are included, not silently dropped)

Revise if needed before saving.

### Step 5: Save Output

Write to `output/diagram-TYPE-YYYY-MM-DD.md`:

```markdown
---
title: "Concept Map — YYYY-MM-DD"
type: diagram
generated: YYYY-MM-DD
scope: whole-wiki | article-name | category-name
---

# Concept Map

\`\`\`mermaid
graph TD
    %% Concept Map — YYYY-MM-DD
    ArticleA["Article A"] --> ArticleB["Article B"]
    ArticleA --> ArticleC["Article C"]
    subgraph Category1
        ArticleB
        ArticleD["Article D"]
    end
\`\`\`

Generated from [[wiki/_index]] on YYYY-MM-DD.
Covers N articles across M categories.
```

Tell the user the file path and that they can open it in Obsidian to see the rendered diagram (requires the Mermaid plugin or Obsidian 1.4+, which has native Mermaid support).

### Step 6: Offer Follow-Ups

After saving, offer:
- "Want a filtered view showing only articles in a specific category?"
- "Want me to add this diagram as an embed in `wiki/overview.md`?"
- "Want a timeline view instead?"

## Common Mistakes

- Generating a 100-node graph for a large wiki -- always offer a filtered view for wikis over 30 articles
- Using markdown links instead of node labels -- all references inside Mermaid blocks are Mermaid syntax, not wikilinks
- Placing the Mermaid file in `wiki/` instead of `output/` -- diagrams are generated artifacts, not wiki articles (unless the user explicitly asks to embed one)
- Forgetting to escape special characters in node labels (parentheses, quotes) -- use double quotes around all labels
