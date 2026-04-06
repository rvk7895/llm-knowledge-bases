---
name: wiki-reflect
description: Synthesize a personal wiki over time — surface patterns, generate monthly or yearly reviews, show how thinking on a topic has evolved, and find decisions whose outcomes are now visible. Triggers on "reflect", "monthly review", "how has my thinking changed", "what patterns emerged", "year in review", or "wiki-reflect".
argument-hint: "monthly [YYYY-MM] | yearly [YYYY] | topic <name> | decisions | patterns"
---

# Personal Wiki Reflection

Reads your compiled personal wiki and synthesizes it across time. Surfaces what changed, what patterns keep recurring, how your thinking evolved, and which past decisions now have visible outcomes.

**Requires the `wiki` skill to have been run first.** Reads from `wiki/` — never from `raw/entries/`.

**First action:** read `wiki/_index.md` and `wiki/_backlinks.json`. If neither exists, tell the user to run `/wiki absorb` first.

---

## Modes

Run with an argument or let Claude infer the right mode from context:

| Mode | Trigger | What it produces |
|------|---------|-----------------|
| `monthly` | "monthly review", "what happened in March" | Narrative summary of one calendar month |
| `yearly` | "year in review", "what happened in 2025" | Annual synthesis across all themes |
| `topic` | "how has my thinking on X changed" | Evolution of one topic across time |
| `decisions` | "which decisions played out" | Past decisions with now-visible outcomes |
| `patterns` | "what patterns keep coming up" | Recurring behavioral or emotional cycles |

---

## Mode: Monthly Review

**Default period:** last calendar month. Override with `YYYY-MM`.

### Step 1: Find the Month's Material

Scan `wiki/_index.md` for articles with `last_updated` or `created` dates in the target month. Also scan `wiki/_absorb_log.json` for entries absorbed during that period.

Collect:
- Articles created this month (new things that entered the person's life)
- Articles significantly updated this month (things that evolved or ended)
- Decisions made this month (`wiki/decisions/`)
- Transitions or events that occurred (`wiki/transitions/`, `wiki/events/`)
- People who appeared prominently (`wiki/people/`)

### Step 2: Read the Material

Read every article identified in Step 1. Follow `[[wikilinks]]` one hop deep when a connection seems significant.

### Step 3: Write the Review

Structure:

```markdown
# [Month YYYY] — Reflection

## What happened
One paragraph. The plain facts of the month — what started, what ended, what shifted.
Not a list. A narrative.

## The through-line
What was the month really about, underneath the surface events?
One theme that connects the scattered facts.

## People
Who mattered this month and how. Direct quotes from entries if they carry weight.

## Decisions made
What was decided and what the reasoning was.

## What's unresolved
Things that started but didn't conclude. Open loops.

## One line
The month in a single sentence.
```

### Step 4: Save

Write to `output/reflect-YYYY-MM.md`. Offer to file it back into the wiki as a `wiki/assessments/` article.

---

## Mode: Yearly Review

**Default:** last calendar year. Override with `YYYY`.

### Step 1: Read All Monthly Reviews First

If monthly reviews exist in `output/` for this year, read them. They're the foundation. Only read raw wiki articles for months without a review.

### Step 2: Read the Big Articles

Articles with the highest backlink counts in `wiki/_backlinks.json` represent the most central things in this person's life. Read the top 15.

Also read:
- All `wiki/eras/` articles updated this year
- All `wiki/transitions/` articles from this year
- All `wiki/decisions/` articles from this year

### Step 3: Write the Year Review

Structure:

```markdown
# [YYYY] — Year in Review

## The arc
The year as a single narrative — what it was about from start to finish.
No bullet points. A story.

## Chapters
The 2-4 distinct phases the year broke into, with approximate dates.
Each gets a paragraph.

## What grew
Projects, relationships, ideas, skills that developed significantly.

## What ended
Things that concluded, relationships that changed, phases that closed.

## Decisions and their outcomes
Decisions made this year with outcomes now visible.
Decisions from prior years whose outcomes became clear this year.

## Patterns that held
Recurring behaviors, emotional cycles, or tendencies that showed up again.

## What surprised you
Things that went differently than expected — based on what the entries actually show,
not what the person predicted.

## One paragraph
The year in a single paragraph. Direct, plain, no qualifiers.
```

### Step 4: Save

Write to `output/reflect-YYYY.md`. Offer to file as `wiki/eras/YYYY.md`.

---

## Mode: Topic Evolution

**Usage:** `/wiki-reflect topic <topic name>`

Show how thinking on a specific topic evolved over time.

### Step 1: Find the Topic

Search `wiki/_index.md` for the topic and its aliases. Find the article and all articles that link to it (`wiki/_backlinks.json`).

### Step 2: Build the Timeline

Read the topic article and all articles that reference it. Extract every mention with its approximate date (from `created`, `last_updated`, or entry source dates in frontmatter).

Order mentions chronologically.

### Step 3: Write the Evolution

Structure:

```markdown
# How thinking on [Topic] evolved

## Where it started
First appearance, first impressions, initial framing.

## How it developed
Chronological narrative of how the understanding deepened, shifted, or complicated.
Use direct quotes from entries at turning points.

## Where it stands now
Current understanding — what solidified, what's still uncertain.

## What changed it
The specific experiences, people, or ideas that shifted the thinking.

## Open questions
What remains unresolved about this topic.
```

### Step 4: Save

Write to `output/reflect-topic-<name>-YYYY-MM-DD.md`. Offer to update the topic's wiki article with a new "Evolution" section.

---

## Mode: Decisions Review

Find past decisions whose outcomes are now visible.

### Step 1: Find All Decisions

Read all articles in `wiki/decisions/`. Extract:
- The decision made
- The date
- The reasoning given at the time
- Any predicted outcomes

### Step 2: Find Outcome Evidence

For each decision, search the rest of the wiki for articles that reference it or were created after it and are related. Look for evidence of:
- The predicted outcome playing out (or not)
- Unexpected consequences
- How the decision is referenced in later entries (regret, relief, neutrally)

### Step 3: Write the Report

For each decision with visible outcomes:

```markdown
## [Decision name] — [Date]

**The decision:** What was decided and why.

**The prediction:** What was expected to follow.

**What actually happened:** Evidence from later wiki articles.

**Verdict:** Did it play out as expected? What wasn't anticipated?
```

Group by: played out as expected / played out differently / outcome still unclear.

### Step 4: Save

Write to `output/reflect-decisions-YYYY-MM-DD.md`.

---

## Mode: Patterns

Surface recurring behavioral or emotional patterns across the whole wiki.

### Step 1: Read Pattern Articles

Read all articles in `wiki/patterns/`, `wiki/tensions/`, and `wiki/philosophies/`.

### Step 2: Cross-Reference

For each pattern article, check `wiki/_backlinks.json` to find how many other articles reference it. High backlink count = pattern that keeps showing up.

Also scan `wiki/decisions/` and `wiki/transitions/` for implicit patterns not yet given their own article — recurring situations where the same type of choice keeps being made.

### Step 3: Write the Pattern Report

```markdown
# Pattern Report — YYYY-MM-DD

## Most active patterns
Patterns with the highest recent activity (articles updated in last 6 months that reference them).
For each: name, brief description, recent evidence, whether it's intensifying or fading.

## Patterns that may have shifted
Patterns with high historical backlinks but low recent activity — possibly resolved or transformed.

## Emerging patterns
Recurring themes in recent articles that don't yet have a pattern article.
Candidate new patterns to create.

## Tensions that remain unresolved
Articles in wiki/tensions/ with continued evidence of conflict.
```

### Step 4: Save

Write to `output/reflect-patterns-YYYY-MM-DD.md`. Offer to create new pattern articles for emerging patterns identified.

---

## Writing Standards

Reflect uses the same tone rules as the `wiki` skill:
- Wikipedia-flat, factual, no editorial voice
- Direct quotes carry the emotional weight — the prose stays neutral
- No em dashes, no peacock words, no rhetorical questions
- Attribution over assertion: "entries from this period describe X as..." not "it was clearly X"

One addition specific to reflection: **be honest about gaps**. If the wiki doesn't have enough material to support a claim about a pattern or outcome, say so explicitly. Don't invent continuity.

---

## Scheduling

To run monthly reviews automatically, use `/schedule`:

```
/schedule "On the 1st of every month, run /wiki-reflect monthly for the previous month and save to output/"
```

Or use `/loop` for more frequent check-ins:

```
/loop 1w /wiki-reflect patterns
```

---

## Common Mistakes

- Reading `raw/entries/` instead of `wiki/` — the wiki is the knowledge base; raw entries are inputs, not the source of truth for reflection
- Writing the review before reading enough articles — read broadly before synthesizing
- Inventing patterns that aren't in the wiki — every claim must trace back to a specific article or entry
- Producing a bullet list instead of a narrative — reflection is synthesis, not a summary; write in paragraphs
- Filing every reflection output back into the wiki automatically — only file if it adds genuinely new understanding; a monthly summary of what's already in the wiki doesn't need to be re-filed
