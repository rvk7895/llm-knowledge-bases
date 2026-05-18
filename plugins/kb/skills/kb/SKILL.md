---
name: kb
description: "Use when compiling raw sources into the wiki, querying the knowledge base, running health checks or lint on the wiki, evolving or improving wiki coverage, or when user says 'compile', 'update the wiki', 'query', 'lint', 'health check', 'evolve', 'what's missing', or 'suggest improvements'. Also triggers on questions that should be answered from the knowledge base."
---

## Overview

Main operating skill for LLM-maintained knowledge bases. Five workflows: **compile**, **query**, **lint**, **evolve**, **reflect**. All opus-orchestrated with model-aware subagent dispatch.

**First action in every invocation:** read `kb.yaml` from the project root. If missing, tell the user to run `kb-init` and stop.

## Report Preferences

After reading `kb.yaml`, extract the `report_preferences:` block. These are free-text prose instructions the user set via `kb-init` or `/kb-preferences`, controlling how outputs are written (audience, register, depth, code_handling, diagrams, self_containment, citations, argument_iteration, notes).

**Apply to all generated prose**:
- **Compile**: preferences shape how new wiki articles are written — register, self-containment, depth of per-article profiles, diagram choices in article bodies.
- **Query**: preferences shape the final answer prose and any new wiki articles auto-evolved from the query.
- **Lint / Evolve / Reflect**: preferences shape the output report prose (lint reports, evolve suggestions), not the mechanical checks themselves.

**If `report_preferences:` is missing**: fall back to factory defaults in `plugins/kb/references/report-style-guide.md` silently. Mention it once in the first output of the session: "No `report_preferences` set — using factory defaults. Run `/kb-preferences init` to customize."

**Per-task overrides.** If the user's current request explicitly contradicts a stored preference, follow the request for this task only. Do NOT modify `kb.yaml` — reflection handles persistence.

### Reflection at end of compile and query

After a compile or query finishes writing its output, run a **lightweight reflection step** — only propose preference updates for **high-confidence signals**:
- User explicitly asked you to "remember" something
- User corrected the same style choice twice in the conversation
- User gave clear feedback directly tied to the output just produced

Follow the **Reflection Protocol** in `plugins/kb/skills/kb-preferences/SKILL.md`. On approval, write updates to `kb.yaml` and log to `.meta/_preference_history.md` with trigger `reflection after /kb compile` (or `/kb query`). Skip silently otherwise.

Do NOT run reflection for lint / evolve / reflect workflows — their output is too mechanical to surface meaningful style signals most of the time. The user can run `/kb-preferences reflect` manually if they want a forced pass.

## Obsidian-Native Formatting

All wiki output MUST use Obsidian-native conventions:

- YAML frontmatter delimited by `---`
- `[[wikilinks]]` for ALL internal links -- NEVER use markdown-style `[text](url)` links for internal references
- `[[wikilinks|display text]]` when the display name differs from the target article
- Tags in frontmatter: `tags: [concept, topic]`
- Aliases in frontmatter: `aliases: [alternate name]`
- Image embeds: `![[image-name.png]]`
- Standard markdown for everything else (headings, lists, bold, code blocks, etc.)

## Graph Hygiene

Every `.md` file in an Obsidian vault becomes a node in the graph. Every internal link — `[[wikilink]]` or markdown-style `[text](file.md)` — becomes an edge. Link syntax does not control the graph. Inventory files (`_index`, `_sources`, `_categories`, `_evolution`, `_tags`) exist as files that link to every article, so they appear as the highest-degree nodes by construction. The fix is to **keep them out of the vault index entirely.**

- **Meta/index files live in `.meta/` at the vault root, not in `wiki/`.** Obsidian ignores any folder whose name starts with `.` (the same mechanism that hides `.obsidian/` itself). Files in `.meta/` do not appear in the graph, search, quick switcher, backlinks, or file explorer. They remain readable and writable by tooling.

  Canonical paths:
  - `.meta/_index.md` — master article index
  - `.meta/_sources.md` — raw-source → article mapping
  - `.meta/_categories.md` — auto-maintained category tree
  - `.meta/_evolution.md` — append-only change log
  - `.meta/_tags.md` — canonical tag vocabulary

  Any new meta/inventory file goes in `.meta/` too, never in `wiki/`.

- **Article-to-article links MUST be `[[wikilinks]]`.** These carry semantic weight and are the intended graph edges.
- **MOC pages MUST use `[[wikilinks]]` and live in `wiki/`.** MOCs are curated, narrative conceptual hubs and belong in the graph.
- **Verify every `[[wikilink]]` target exists before emitting it.** Confirm either (a) the target article already exists in `wiki/**/*.md` (including via its `aliases` frontmatter), or (b) the agent is creating that target in the same compile pass. If neither, do NOT emit a wikilink — use plain text or italics. Never emit a wikilink as a "TODO placeholder." *Why:* dangling wikilinks render as ghost/placeholder nodes in the graph and open an empty editor when clicked.

### On `kb-init` and compile

Ensure `.meta/` exists in the vault root. If the vault contains legacy meta files at `wiki/_index.md`, `wiki/_sources.md`, `wiki/_categories.md`, `wiki/_evolution.md`, or `wiki/_tags.md`, move them to `.meta/` with the same filename, and make sure no `wiki/**/*.md` articles reference them (they shouldn't — meta files reference articles, not the other way around).

`.meta/` being a dotfolder is load-bearing. Do NOT rename it to `meta/` or `_meta/` — Obsidian would then index it.

**Past mistakes recorded here so future agents don't repeat them:**
1. Converting meta-file wikilinks to markdown-style links in the belief that this reduces graph degree — it does not; Obsidian resolves both styles as edges.
2. Relying on the graph-view `search` filter in `.obsidian/graph.json` — it hides files from the graph only, not from search/backlinks/quick-switcher.
3. Using `userIgnoreFilters` in `.obsidian/app.json` — this was partially correct (it does hide files from the index), but leaves the files visible in the file explorer and depends on the user not clearing the filter. The `.meta/` dotfolder is the structural fix: Obsidian cannot index it regardless of user settings.

## Tag Hygiene

Tags are for cross-cutting classification, not restating the title. Without a controlled vocabulary, tags drift (`reasoning` vs `reasoning-models` vs `cot` vs `chain-of-thought`) and lose their value as navigation.

- **Use a namespaced, controlled vocabulary.** Every tag MUST fall into one of these namespaces:
  - `domain/*` — broad subject area (e.g., `domain/reasoning`, `domain/rl`, `domain/alignment`, `domain/interpretability`)
  - `method-family/*` — class of technique (e.g., `method-family/policy-gradient`, `method-family/preference-optimization`)
  - `status/*` — mirrors the `status` frontmatter field (`status/seedling`, `status/budding`, `status/evergreen`)
  - `moc` — the existing bare tag for Map of Content pages (preserved for continuity)
- **Tags must not restate the title.** If an article is titled "GRPO," do NOT tag it `grpo`. Tag it `domain/rl`, `method-family/policy-gradient`. The title and aliases already handle that lookup dimension.
- **Reuse before inventing.** Before adding a new tag to any article, read `.meta/_tags.md` (the canonical tag index — see Index Maintenance) and pick an existing tag if it fits. Invent a new tag only when no existing tag captures the dimension, and when you do, add it to `.meta/_tags.md` in the same pass.
- **Lint must flag tag drift** — singletons (tags used on only one article), near-duplicates (different surface forms of the same concept), and any tag that violates the namespace convention. See Workflow 3.

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
| Media type detection | haiku subagents | none |
| Image/diagram description | sonnet subagents | opus reviews |
| MOC generation | sonnet subagents | opus reviews |

## Entity Type System

Every wiki article MUST declare an entity type in frontmatter. Types:

| Type | Description | Examples |
|---|---|---|
| `concept` | Abstract idea, principle, phenomenon | Chain-of-Thought Reasoning, Superposition |
| `method` | Algorithm, technique, procedure | GRPO, DPO, Flash Attention |
| `model` | Specific ML model or system | DeepSeek-R1, Tulu 3 |
| `tool` | Software, library, framework | PyTorch, vLLM |
| `benchmark` | Evaluation dataset or metric | MATH, SWE-Bench |
| `dataset` | Training/evaluation data | OpenWebText, RedPajama |
| `person` | Researcher, author | (rare — only if heavily referenced) |
| `phenomenon` | Observed behavior without explanation | Attention Sinks, Reward Hacking |
| `moc` | Map of Content navigation page | RL & Alignment MOC |

## Maturity Model

Every article MUST declare a maturity status in frontmatter:

| Status | Criteria | Action |
|---|---|---|
| `seedling` | Initial extraction from one source. Under 300 words or single-perspective. | Auto-assigned on creation. Lint flags seedlings older than 30 days. |
| `budding` | Multiple sources integrated. Key relationships mapped. 300-800 words. | Promoted when second source adds information or article is enriched via evolve/query. |
| `evergreen` | Comprehensive coverage. 3+ sources. Well-linked. Relationships section complete. 800+ words. | Promoted by opus during compile/evolve when criteria are met. |

Status transitions are unidirectional: seedling -> budding -> evergreen. The compile and auto-evolve workflows automatically check and promote status when criteria are met.

## Workflow 1: Compile

**Trigger:** User adds files to `raw/` and says "compile" or "update the wiki".

**Apply `report_preferences` from `kb.yaml` to all article prose** (register, self_containment, depth per profile section, diagrams). See the Report Preferences section above. Run lightweight reflection at the end.

### Incremental Detection

1. Read `.meta/_index.md` (lists every raw source with its last-compiled hash)
2. Scan `raw/` recursively with Glob
3. Diff against index: identify **new**, **changed**, and **deleted** sources
4. Only process what changed -- never recompile the entire `raw/` folder

### Omni-Document Type Detection

Before extraction, classify each new/changed file using the media routing table:

| File Type | Detection | Raw Directory | Extraction Method |
|---|---|---|---|
| PDF (.pdf) | Extension | `raw/papers/` | Read tool (supports PDF) -> sonnet extraction |
| Markdown/Text (.md, .txt) | Extension | `raw/articles/` | Read tool -> sonnet extraction |
| Image (.png, .jpg, .svg, ...) | Extension | `raw/images/` | Read tool (supports images) -> vision description |
| Video (.mp4, .mkv, ...) | Extension | `raw/transcripts/` | ffmpeg audio extract -> Whisper transcription -> sonnet |
| Audio (.mp3, .wav, ...) | Extension | `raw/transcripts/` | Whisper transcription -> sonnet |
| Slides (.pptx) | Extension | `raw/articles/` | python-pptx extraction -> sonnet |
| Notebook (.ipynb) | Extension | `raw/repos/` | Read tool (supports notebooks) -> sonnet |
| Code repo (directory with .git) | Directory check | `raw/repos/` | Tree + key files -> sonnet |
| YouTube URL | URL pattern | `raw/transcripts/` | yt-dlp transcript -> sonnet |
| arXiv URL | URL pattern | `raw/papers/` | Download PDF -> process as PDF |
| X/Twitter URL | URL pattern | `raw/notes/` | Smaug or manual paste -> sonnet |
| Web URL | URL pattern | `raw/articles/` | WebFetch -> sonnet |
| Dataset (.csv, .json, .parquet) | Extension | `raw/datasets/` | Schema + sample -> sonnet |

**Detection heuristic:** Run this check for each file:

```python
# Pseudocode — the agent uses its tools, not this script directly
if is_url(source):
    match url patterns -> youtube, arxiv, tweet, github_repo, web_page
elif is_directory(source):
    check for .git, package.json, pyproject.toml -> repository
else:
    match extension -> pdf, text, image, video, audio, slides, notebook, dataset
```

### Per-Type Extraction Procedures

**PDF papers:**
1. Read with Read tool (specify pages for large PDFs)
2. Extract: title, authors, year, abstract, key contributions, methodology, results
3. Extract: figure/table descriptions, key equations (as LaTeX or plain text)
4. Extract: references to other works (may link to existing wiki articles)
5. Determine entity type: usually `method`, `model`, or `concept`

**Images and diagrams:**
1. Read with Read tool (multimodal — will render the image)
2. Describe: what is shown (architecture diagram, chart, screenshot, poster, photo)
3. Extract any visible text (OCR via vision capability)
4. Identify concepts depicted and link to wiki articles
5. Create article with `![[image-name.png]]` embed
6. Set `media_type: image` in frontmatter

**Video files (local):**
1. Check if Whisper is available: `which whisper 2>/dev/null`
2. If available: extract audio `ffmpeg -i input.mp4 -vn -acodec pcm_s16le -ar 16000 /tmp/audio.wav` then `whisper /tmp/audio.wav --model base --output_format txt`
3. If NOT available: tell user "Whisper not installed. Please provide a transcript, or install: pip install openai-whisper"
4. Process transcript as text

**YouTube URLs:**
1. Try: `yt-dlp --write-auto-sub --sub-lang en --skip-download --output "/tmp/%(id)s" "<url>"` to get subtitles
2. If yt-dlp unavailable: try fetching transcript via WebFetch from a transcript service
3. If all fail: ask user to paste transcript
4. Segment transcript by topic, preserve timestamps
5. Set `media_type: video` in frontmatter, include channel and video title

**Audio files (podcasts, recordings):**
1. Same Whisper pipeline as video (skip ffmpeg if already audio)
2. Attempt speaker diarization via context clues (if multiple speakers obvious from content)
3. Create structured notes with `[HH:MM:SS]` timestamps for key insights
4. Set `media_type: audio` in frontmatter

**Slides (.pptx):**
1. Try extracting with python-pptx: `python -c "from pptx import Presentation; ..."` or use Read tool
2. Per slide: extract title, body text, speaker notes
3. For slides with charts/diagrams: describe the visual
4. Structure article following the presentation narrative
5. Set `media_type: slides` in frontmatter

**Code repositories:**
1. Read README.md for overview
2. Bash: `find <repo> -name "*.py" -o -name "*.js" -o -name "*.ts" | head -30` for file listing
3. Read key entry points, configuration files
4. Extract: purpose, architecture, key APIs, dependencies, notable patterns
5. Set `type: tool` and `media_type: repository` in frontmatter

**Datasets:**
1. For CSV: `head -5 <file>` for schema, `wc -l <file>` for row count
2. For JSON: read first few entries, extract schema
3. For Parquet: `python -c "import pandas as pd; df = pd.read_parquet('<file>'); print(df.dtypes); print(len(df))"`
4. Document: what the data represents, schema, size, potential uses
5. Set `type: dataset` in frontmatter

**Notebooks (.ipynb):**
1. Read with Read tool (supports notebooks — shows cells + outputs)
2. Extract: narrative, experiments, results, conclusions
3. Extract: key code patterns and findings
4. Create article focusing on insights, not code reproduction

### Per-File Extraction (sonnet subagents)

For each new or changed source, dispatch a sonnet subagent with:

1. The raw source content (read via appropriate method for type)
2. The current `.meta/_index.md` (so it knows existing articles)
3. Instructions to extract:
   - Key concepts, claims, entities, and relationships
   - Entity type classification
   - Connections to existing wiki articles
   - Suggested category placement

The subagent returns a structured extraction object.

### Opus Orchestration

After extractions complete, opus:

5. **New concepts** -> create wiki article with proper frontmatter (including `type` and `status: seedling`)
6. **Existing concepts** -> update and enrich the article with new information; check if status should promote (seedling -> budding if second source, budding -> evergreen if 3+ sources and 800+ words)
7. Add source backlinks to every article touched
8. Auto-categorize articles into folders based on topic
9. **Duplicate detection** -> if two sources discuss the same concept, merge into one article (never create duplicates)
10. Check if any category now has 5+ articles without a MOC page; if so, create one (see MOC format below)
11. **Link verification pass.** Before finalizing any article, scan its body for every `[[wikilink]]` and confirm the target exists in `wiki/**/*.md` or is being created in this same compile pass. Replace any dangling link with plain text (or a sentence that doesn't assert a link). Never leave a wikilink as a placeholder — dangling links create ghost nodes in the graph.
12. **Stub guard.** Reject any article under 100 words. If the extraction would produce a stub, prefer merging its content into an existing article on a broader topic, or skip creation entirely. Articles created with `status: seedling` MUST still meet the 100-word floor — the single-source criterion is about provenance, not length. Exception: a genuinely atomic entry (e.g., a one-line benchmark definition) may go shorter, but opus must explicitly justify it in the article body.

### Wiki Article Format

```markdown
---
title: "Concept Name"
aliases: [alternate name, abbreviation]
type: method
status: seedling
tags: [domain/rl, method-family/policy-gradient]
sources:
  - "[[raw/articles/source-file.md]]"
media_type: text
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

# Concept Name

Core explanation of the concept — what it is, why it matters, one-paragraph summary.

## Mechanism

How it works in detail. Include key equations, algorithms, or procedures.

## Key Findings

Numbered or bulleted list of the most important results, claims, or insights from sources.

## Limitations

Known weaknesses, failure modes, or caveats.

## Relationships

- **Predecessor:** [[Foundation Concept]] — what this builds on
- **Successor:** [[Improved Method]] — what improved on this
- **Contrasted with:** [[Alternative Approach]] — key differences
- **Applied in:** [[Model or System]] — where this is used
- **Builds on:** [[Underlying Theory]] — theoretical foundation
- **Related:** [[Adjacent Concept]] — conceptual connection

## Sources

- [[raw/articles/source-file.md]] — key claims extracted from this source
```

### Map of Content (MOC) Format

When a category accumulates 5+ articles, create a MOC page:

```markdown
---
title: "Category Name"
type: moc
status: evergreen
tags: [moc, category-name]
created: 2026-04-07
updated: 2026-04-07
---

# Category Name

Brief overview of this knowledge domain and why it matters.

## Core Concepts

- [[Concept A]] — one-line summary
- [[Concept B]] — one-line summary

## Methods & Techniques

- [[Method X]] — one-line summary
- [[Method Y]] — one-line summary

## Models & Systems

- [[Model Z]] — one-line summary

## Key Relationships

Brief narrative of how concepts in this category connect to each other
and to concepts in other categories.

## Open Questions

- Questions this category's articles don't yet answer
- Gaps that could be filled with more research
```

### Index Maintenance

After every compile, update these files:

- **`.meta/_index.md`** -- master list: article name, one-line summary, compiled-from hash
- **`.meta/_sources.md`** -- mapping from raw sources to wiki articles they contributed to
- **`.meta/_categories.md`** -- auto-maintained category tree reflecting folder structure, with article counts
- **`.meta/_evolution.md`** -- append-only log: `date | trigger | action | articles affected`. Create if missing.
- **`.meta/_tags.md`** -- canonical tag vocabulary. A flat list of every allowed tag with a one-line description, grouped by namespace (`domain/*`, `method-family/*`, `status/*`, plus `moc`). Any new tag introduced in this compile pass must be appended here with its description. Agents consult this file before inventing a new tag. Create if missing.

**Link-style inside `.meta/` files is free.** Because `.meta/` is ignored by Obsidian, links inside these files do not create graph edges regardless of syntax. Pick whatever is easiest for tooling to parse — typically plain markdown-style links with paths relative to the vault root (e.g., `[GRPO](wiki/rl-alignment/GRPO.md)`) or just article titles in table cells. Do not waste cycles converting between styles.

## Workflow 2: Query

**Trigger:** User asks a question about the knowledge base.

**Apply `report_preferences` from `kb.yaml` to the final answer prose and to any auto-evolved article updates.** See the Report Preferences section above. Run lightweight reflection at the end (after the answer is delivered; independent of the auto-evolve subagent).

### Step 1: Ask Query Depth

Present three options:

- **Quick** -- wiki-only. Read indexes, find relevant articles, synthesize an answer. No external calls.
- **Standard** -- wiki + web. Start from wiki, use WebSearch/WebFetch to fill gaps.
- **Deep** -- full research pipeline. Invoke `/research` for outline, `/research-deep` dispatches sonnet subagents, opus orchestrates and verifies.

### Step 2: Execute at Chosen Depth

Run the appropriate pipeline. For Standard and Deep, clearly mark which information came from the wiki and which came from external sources.

### Step 2.5: Persist Web Sources (Standard and Deep only)

See **Auto-Capture Web Research** section below. Any web pages fetched via WebSearch/WebFetch that contributed to the answer must be saved as raw documents.

### Step 3: Render Output

Format the answer according to `kb.yaml` output preferences (markdown, Marp, matplotlib, etc.) and save to `output/`.

### Step 4: Auto-Evolve (background opus subagent)

After delivering the answer to the user, **immediately dispatch a background opus subagent** (via Agent tool with `run_in_background: true`). Do NOT ask the user for permission — this runs silently.

The subagent receives:
- The original query
- The full response (including sources used and depth level)
- The current `.meta/_index.md`

The subagent decides and acts:

1. **New knowledge?** — Did the response contain information not already in the wiki? (External data from Standard/Deep queries, new connections between concepts, corrections to existing articles, newly surfaced entities or claims)
2. **If no** — exit silently. Log nothing. The query was just a read.
3. **If yes** — act:
   - **New concept** → create a wiki article following the standard article format. Include `type`, `status: seedling`, and all required frontmatter fields.
   - **Enrichment** → update existing articles with new information, add source backlinks, check for status promotion
   - **Correction** → update articles where the response contradicted or superseded wiki content
   - **New connections** → add `[[wikilinks]]` between articles that the query revealed are related
   - Update `.meta/_index.md`, `.meta/_sources.md`, `.meta/_categories.md`
   - Append a one-line entry to `.meta/_evolution.md`: `YYYY-MM-DD | query | action taken | articles affected`

**Constraints:**
- Never delete or downgrade existing content — only add or refine
- Never create stub articles under 100 words — if there isn't enough to say, add the knowledge to an existing article instead
- Quick-depth queries rarely produce new knowledge — the subagent should almost always exit silently for these
- The subagent must re-read any article it plans to modify (not rely on cached state)
- Always include `type` and `status` fields in any new article frontmatter

## Workflow 3: Lint

**Trigger:** "health check", "lint the wiki", or invoked via `/loop` or `/schedule`.

### Phase 1: Automated Analysis

Run the graph health script if available:

```bash
python plugins/kb/scripts/graph_health.py "<vault_path>" --json --fix-suggestions
```

If the script is not available, perform the checks manually via subagents.

### Checks (sonnet subagents in parallel)

1. **Broken links** -- `[[wikilinks]]` pointing to non-existent articles
2. **Orphan articles** -- wiki articles with zero inbound links
3. **Orphan sources** -- files in `raw/` that were never compiled
4. **Stale articles** -- source file changed since the article was last compiled
5. **Consistency** -- conflicting claims across different articles
6. **Missing backlinks** -- links that should be bidirectional but aren't
7. **Sparse articles** -- articles below ~200 words
8. **Missing frontmatter** -- articles without `type` or `status` fields
9. **Stale seedlings** -- articles with `status: seedling` older than 30 days
10. **MOC gaps** -- categories with 5+ articles but no MOC page
11. **Potential links** -- articles mentioning concepts they don't link to
12. **Meta files in `wiki/` instead of `.meta/`** -- any `_*.md` file under `wiki/` should be moved to `.meta/` so Obsidian does not index it. Flag as a fix.
13. **Tag vocabulary violations** -- tags not matching the namespaced scheme (`domain/*`, `method-family/*`, `status/*`, `moc`), or any tag not listed in `.meta/_tags.md`.
14. **Article title shadow-tags** -- an article tagged with the slug of its own title (e.g., article "GRPO" tagged `grpo`). Tags are for cross-cutting classification.
15. **Under-100-word stubs** -- articles whose body is under 100 words and that never passed the stub-guard bar.

### Phase 2: Graph Metrics

Report these metrics in the lint output:

| Metric | Target | What It Means |
|---|---|---|
| Orphan rate | < 5% | Articles nobody links to |
| Broken link rate | < 2% | Links pointing to nothing |
| Avg outgoing links | >= 5 | How well-connected articles are |
| Bidirectional rate | >= 50% | Two-way relationship coverage |
| Evergreen rate | >= 30% | Maturity of the knowledge base |
| Type diversity | >= 4 types | Variety of entity types |
| Cross-cluster links | report count | Knowledge integration across categories |

### Opus Collation

Opus collates all findings, filters false positives, and prioritizes by severity:
- **Critical**: broken links, missing articles referenced by 3+ others
- **Warning**: orphans, sparse articles, stale seedlings, missing frontmatter fields
- **Info**: potential links, MOC gaps, missing backlinks

### Output

Save to `output/lint-YYYY-MM-DD.md`. Include:
1. Graph metrics summary table
2. Issues grouped by severity with suggested fixes
3. Comparison to previous lint if available (improvement/regression)

Ask the user if they want auto-fix applied for mechanical fixes (adding missing frontmatter fields, creating MOC pages, fixing broken links to articles that exist under different names).

## Workflow 4: Evolve

**Trigger:** "evolve the wiki", "suggest improvements", or "what's missing".

### Process

1. Opus reads `.meta/_index.md`, `.meta/_categories.md`, and samples articles across categories
2. Dispatch sonnet subagents to analyze article clusters for:
   - **Gaps** -- concepts referenced in articles that lack their own article (extract from broken links + mention analysis)
   - **Connections** -- cross-category relationships not yet explored
   - **Missing data** -- claims that could be verified or enriched via web search
   - **Questions** -- interesting unanswered questions surfaced by the existing content
   - **MOC candidates** -- categories needing Map of Content pages
   - **Maturity promotions** -- articles that qualify for status upgrade
   - **Bridging articles** -- concepts that connect two otherwise separate clusters
3. Opus collates, deduplicates, and ranks suggestions by value:
   - High value: fills a gap referenced by 3+ articles, creates a bridging concept
   - Medium value: enriches existing articles, adds cross-category links
   - Low value: cosmetic improvements, minor reorganization
4. Present as a numbered list with brief rationale for each
5. User picks items -> Claude executes via Compile, Query, or web search as appropriate

### Auto-Evolve MOC Generation

If the evolve analysis identifies categories with 5+ articles and no MOC, **automatically create MOC pages** (don't just suggest — do it). MOCs are low-risk, high-value additions that improve navigability.

## Workflow 5: Reflect

**Trigger:** "reflect on the wiki", "what have we learned", "knowledge audit", or after every 10th compile.

This is the meta-improvement workflow inspired by autoresearch. It evaluates the knowledge base against quality rubrics and proposes structural improvements.

### Process

1. Run the evaluation script if available:
   ```bash
   python experiments/evaluate.py "<vault_path>" --json
   ```
2. Analyze the scores across all rubric categories
3. Identify the 3 lowest-scoring areas
4. For each weakness, propose a specific action:
   - If `compile` scores low: improve extraction prompts or add missing frontmatter fields
   - If `graph_health` scores low: fix broken links, connect orphans, create MOCs
   - If `knowledge_architecture` scores low: refactor articles, improve typing
5. Present the analysis and proposed actions to the user
6. If user approves, execute the improvements
7. Re-run evaluation to measure improvement
8. Log results to `experiments/logs/experiment_log.md`

## Handling X/Twitter Links

When the user pastes an `x.com` or `twitter.com` URL and wants it added to the knowledge base:

### Step 1: Check if Smaug is configured

Read `kb.yaml` and look for `integrations.smaug.path`. If it exists, verify the path is still valid:

```bash
test -d "<smaug_path>" && test -f "<smaug_path>/smaug.config.json" && echo "smaug-ready" || echo "smaug-missing"
```

If `integrations.smaug` is not in `kb.yaml`, try finding it:

```bash
which bird 2>/dev/null && find ~ -maxdepth 4 -name "smaug.config.json" -type f 2>/dev/null | head -1
```

If found, **save the path to `kb.yaml`** under `integrations.smaug.path` so future sessions don't need to search again.

### Step 2a: Smaug IS available

1. `cd` to the Smaug path from `kb.yaml`
2. Extract the tweet ID from the URL (the numeric part after `/status/`)
3. Run: `npx smaug fetch <tweet_id>` then `npx smaug process`
4. Smaug outputs markdown with frontmatter to its `knowledge/` directory
5. Copy the output to `raw/articles/` and trigger Compile

### Step 2b: Smaug is NOT available

Tell the user you cannot directly fetch X/Twitter content, then present these options:

1. **Install Smaug** (recommended) — `git clone https://github.com/alexknowshtml/smaug && cd smaug && npm install`, then `npx smaug setup` to configure X session cookies (`auth_token` + `ct0` from browser DevTools → Cookies → x.com). After setup, **save the install path to `kb.yaml`** under `integrations.smaug.path`. Note: uses session cookies, technically violates X TOS but practical risk for personal read-only use is very low.
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

## Handling YouTube URLs

When the user provides a YouTube URL:

1. Try to fetch transcript: `yt-dlp --write-auto-sub --sub-lang en --skip-download --output "/tmp/%(id)s" "<url>" 2>&1`
2. If yt-dlp succeeds: read the subtitle file, clean timestamps, process as text
3. If yt-dlp not installed: try `pip install yt-dlp` then retry
4. If still fails: ask user to paste transcript manually, or try fetching from transcript service via WebFetch
5. Extract video metadata: `yt-dlp --dump-json "<url>" 2>/dev/null | python -c "import sys,json; d=json.load(sys.stdin); print(f'Title: {d[\"title\"]}\nChannel: {d[\"channel\"]}\nDuration: {d[\"duration\"]}s')"` if yt-dlp available
6. Save transcript to `raw/transcripts/yt-<video_id>.md` with frontmatter including title, channel, URL
7. Process through standard compile pipeline

## Auto-Capture Web Research

**Applies to ALL workflows** — Compile, Query (Standard/Deep), Evolve — any time WebSearch or WebFetch is used to gather context.

**Config:** `auto_capture_web` in `kb.yaml` (default: `true`). If `false`, skip entirely.

### When it triggers

Any time the skill fetches a web page to help with its work:
- **Compile:** searching for background context to understand a source (e.g., looking up papers referenced in a YouTube video, finding context for a concept mentioned in a PDF)
- **Query (Standard/Deep):** fetching external information to supplement wiki content
- **Evolve:** researching to fill identified gaps

### What to capture

Only pages whose content **actually contributed** to the output. Do NOT save:
- Search result listing pages
- Pages that were fetched but contained nothing useful
- Pages that failed to load or returned errors

### Procedure

For each contributing web page:

1. **Dedup check** — search `raw/` for the URL:
   ```bash
   grep -rl "<url>" raw/ 2>/dev/null | head -1
   ```
   If found, skip saving — just reference the existing file as a source.

2. **Save** — if the URL is new, write to `raw/articles/<slug>.md` using the page title as a natural filename (lowercase, hyphens). Frontmatter:
   ```markdown
   ---
   title: "Page Title"
   url: "https://example.com/article"
   fetched: YYYY-MM-DD
   media_type: web_article
   ---

   [fetched page content]
   ```

3. **Continue** — the saved raw file will be compiled into wiki articles either:
   - Immediately by the auto-evolve subagent (for queries)
   - On the next `/kb compile` run (for compile-time context searches)

### Example

User provides a YouTube video about RLHF. During compile, the skill searches the web for "proximal policy optimization LLM training" to understand a reference in the video. It finds and reads a blog post at `https://example.com/ppo-explained`. That blog post contributed to understanding the video → save it to `raw/articles/ppo-explained.md` with the URL in frontmatter.

## Common Mistakes

- Running compile on the entire `raw/` folder when only a few files changed -- always use incremental detection
- Using markdown-style links `[text](url)` instead of `[[wikilinks]]` for internal references
- Skipping index updates (`.meta/_index.md`, `.meta/_sources.md`, `.meta/_categories.md`) after compile
- Asking the user whether to file query results -- auto-evolve handles this silently in the background
- Defaulting to Deep query depth for simple factual questions -- try Quick first
- Forgetting to dispatch the auto-evolve subagent after a query -- it must always run, even if it usually exits silently
- Refusing to process an X/Twitter link just because Smaug isn't installed -- always offer alternatives and accept manual paste
- Trying to use WebFetch on X/Twitter URLs -- it always fails due to auth walls, don't bother
- **Creating articles without `type` or `status` fields** -- every article MUST have both
- **Creating duplicate articles** for the same concept from different sources -- merge into one
- **Ignoring media type** -- every non-text source should have `media_type` in frontmatter
- **Skipping MOC creation** when a category grows past 5 articles
- **Not promoting article status** when enrichment criteria are met
- **Saving every web fetch as a raw document** -- only save pages that actually contributed to the output
- **Re-saving a URL already in raw/** -- always grep for the URL first to dedup
- **Forgetting to save web research** -- if `auto_capture_web` is on and a web page helped produce the output, it must be saved
- **Forgetting that every .md file inside the vault is a graph node** -- meta files appear in the graph because they exist as files there. Link style does not matter; both `[[wikilinks]]` and `[text](file.md)` become graph edges. The fix is to keep meta files in `.meta/` (a dotfolder at the vault root), which Obsidian does not index at all.
- **Placing meta files under `wiki/`** -- `wiki/_index.md`, `wiki/_anything.md` etc. will always be graph nodes. Meta files belong in `.meta/`.
- **Renaming the dotfolder to `meta/`, `_meta/`, or similar** -- the leading dot is what makes Obsidian ignore it. Non-dot folders are indexed.
- **Emitting `[[wikilinks]]` to articles that don't exist yet and aren't being created in the same pass** -- produces ghost nodes in the graph and opens empty editors when clicked. Use plain text or italics instead.
- **Creating articles shorter than 100 words** -- merge into a broader article instead; the seedling status does not exempt an article from the length floor
- **Tagging articles with the slug of their own title** -- tags are for cross-cutting classification (`domain/*`, `method-family/*`), not restating what the title already says
- **Inventing new tags ad hoc instead of reusing `.meta/_tags.md` vocabulary** -- always consult the tag index first; add to `.meta/_tags.md` only when no existing tag fits
