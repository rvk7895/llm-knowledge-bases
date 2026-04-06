---
name: kb-connect
description: "Use when the user wants to discover hidden connections between wiki articles, find relationships between concepts, surface unexpected links, or when they say 'connect the wiki', 'find connections', 'what's related', 'discover links', or 'relationship map'."
---

## Overview

Scans the entire wiki to surface hidden relationships between articles -- concepts that share ideas but aren't yet cross-linked, articles that contradict each other, and clusters of related knowledge that could be synthesized into new articles.

**First action:** read `kb.yaml` for paths. If missing, tell the user to run `kb-init` and stop.

## Model Strategy

| Task | Executor |
|------|----------|
| Scan article index and sample content | haiku subagents |
| Identify shared concepts across article pairs | sonnet subagents |
| Detect contradictions and conflicts | sonnet subagents |
| Rank and synthesize findings | opus |

## Workflow

### Step 1: Build the Connection Map

1. Read `wiki/_index.md` to get all article names and one-line summaries
2. Dispatch **haiku** subagents to extract a structured concept list from each article:
   - Key terms, entities, and named concepts
   - Existing `[[wikilinks]]` already present
   - Claims that could be verified or contradicted elsewhere
3. Return per-article concept sets to opus

### Step 2: Find Relationships (sonnet subagents in parallel)

Dispatch **sonnet** subagents across article clusters to identify:

**Shared concepts** -- two or more articles reference the same term, person, or idea but don't link to each other

**Contradictions** -- articles make conflicting claims about the same fact (dates, numbers, causes, outcomes)

**Implied articles** -- a concept is referenced in `[[wikilinks]]` across multiple articles but has no dedicated page yet

**Synthesis opportunities** -- a cluster of 3+ articles on related topics that together could support a new higher-level overview article

**Hierarchical gaps** -- a broad topic article exists but lacks links to its sub-topics, or sub-topics exist without linking to their parent

### Step 3: Opus Collation and Ranking

Opus collates all findings, deduplicates, and ranks by value:

1. **Critical** -- contradictions between articles (factual conflicts)
2. **High** -- implied articles that are referenced but missing
3. **Medium** -- shared concepts with no cross-links
4. **Low** -- synthesis opportunities and hierarchy gaps

### Step 4: Present Findings

Output a report to `output/connections-YYYY-MM-DD.md` in this format:

```markdown
# Connection Report — YYYY-MM-DD

## Contradictions (fix these)
- [[Article A]] claims X, [[Article B]] claims Y — same event, conflicting dates
  → Suggested fix: verify source, update one or both articles

## Missing Articles (create these)
- "Concept X" is referenced in [[A]], [[B]], [[C]] but has no page
  → Suggested fix: run `/kb compile` with a stub or ask Claude to draft it

## Missing Cross-Links (quick wins)
- [[Article A]] and [[Article B]] both discuss "Y" but don't link to each other
  → Suggested fix: add [[Article B]] to the Relationships section of [[Article A]] and vice versa

## Synthesis Opportunities
- [[A]], [[B]], [[C]] form a coherent cluster around "Topic X"
  → Suggested fix: create a new overview article "Topic X" that synthesizes all three
```

### Step 5: Act on Findings

After presenting the report, ask:

**"Want me to apply any of these? I can:"**
- Add missing cross-links automatically (quick, non-destructive)
- Draft missing articles and add them to the compile queue
- Flag contradictions for your review (Claude won't auto-resolve factual conflicts)
- Draft synthesis articles for clusters you approve

Apply only what the user approves.

## Obsidian-Native Output

All new links must use `[[wikilinks]]` format. Never use markdown-style links for internal references. New articles created from this workflow follow the standard article format with YAML frontmatter.

## Common Mistakes

- Auto-resolving contradictions without user review -- always flag, never silently pick a side
- Treating every shared word as a meaningful connection -- filter for substantive concept overlap, not keyword coincidence
- Creating synthesis articles without user approval -- suggest, don't act unilaterally
- Running this on a wiki with fewer than 10 articles -- connections only emerge at scale; recommend building up the wiki first
