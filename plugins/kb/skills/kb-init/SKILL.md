---
name: kb-init
description: "Use when setting up a new knowledge base, bootstrapping an Obsidian vault, or when user says 'init kb', 'new knowledge base', 'create kb', or 'setup vault'. Triggers on any request to initialize or scaffold a knowledge base project."
---

## Overview

One-time (or re-runnable) setup that bootstraps a knowledge base project as an Obsidian vault with full omni-document support.

## Prerequisites

Obsidian must be installed.

## Plugin Setup

After confirming the vault path, run the bundled setup script via Bash tool:

```bash
# Interactive — asks user about each optional plugin
bash plugins/kb/setup.sh /path/to/vault

# Install everything
bash plugins/kb/setup.sh /path/to/vault --all

# Install specific plugins only
bash plugins/kb/setup.sh /path/to/vault --only dataview,obsidian-git
```

**Required plugins** (always installed):
- **Dataview** — query wiki articles like a database
- **Obsidian Git** — auto-backup vault to git

**Optional plugins** (user chooses interactively):
- **Kanban** — track wiki tasks on boards
- **Outliner** — better list editing for article drafts
- **Tag Wrangler** — rename and merge tags across the wiki
- **Local Images Plus** — download and store remote images locally

**Browser extension** (printed as manual step):
- **Web Clipper** — clip web articles into `raw/`

The script is idempotent — safe to re-run. If Obsidian is open, tell the user to restart it.

## Flow

### 1. Vault Setup

Ask: create a new vault or use an existing directory? If new, scaffold `.obsidian/`. If existing, verify it exists.

### 2. Source Gathering Strategy

Ask how the user wants to get raw data in:

- **Self-serve** -- User drops files in `raw/`. Per-type guidance:
  - Web articles: Obsidian Web Clipper extension -> `raw/articles/`
  - Academic papers (PDF): Download to `raw/papers/`
  - GitHub repos: Clone/snapshot to `raw/repos/`
  - Local markdown/text: Copy to `raw/notes/`
  - Images/diagrams/posters: Place in `raw/images/`
  - YouTube transcripts: Transcript tool -> `raw/transcripts/`
  - Podcast episodes: Audio files -> `raw/transcripts/`
  - Slide decks (.pptx): Place in `raw/articles/`
  - Datasets (CSV, JSON): Place in `raw/datasets/`
  - Jupyter notebooks: Place in `raw/repos/`
  - X/Twitter posts: Via [Smaug](https://github.com/alexknowshtml/smaug) (optional) -> `raw/articles/`
- **Assisted** -- Claude fetches via WebFetch, Bash (clone repos, download PDFs, pull transcripts, fetch YouTube subtitles)
- **Mixed** -- Some of each

### 3. Output Format Preferences

Ask which formats the user wants: markdown (always on), Marp slides, matplotlib charts, HTML, CSV, Excalidraw, other. Use an extensible pattern so new formats can be added later.

### 4. Maintenance Cadence

Inform about options:
- Daily/hourly: `/loop` (e.g., `/loop 1d kb lint`)
- Weekly/monthly: `/schedule`
- Manual: just ask anytime

### 5. Check Optional Dependencies

Detect and report availability of optional tools that enhance the system:

```bash
# Check each tool
which whisper 2>/dev/null && echo "whisper: available" || echo "whisper: not installed (needed for audio/video transcription)"
which ffmpeg 2>/dev/null && echo "ffmpeg: available" || echo "ffmpeg: not installed (needed for audio extraction from video)"
which yt-dlp 2>/dev/null && echo "yt-dlp: available" || echo "yt-dlp: not installed (needed for YouTube transcript fetching)"
python -c "import pptx" 2>/dev/null && echo "python-pptx: available" || echo "python-pptx: not installed (needed for slide extraction)"
python -c "import pandas" 2>/dev/null && echo "pandas: available" || echo "pandas: not installed (needed for dataset analysis)"
```

Report which media types are fully supported vs. which need manual input. Don't force installation — just inform.

### 5.5 Gather Report Preferences

Reports, wiki compilations, and query answers are written very differently depending on who the reader is, how much depth they want, and how they want code and references handled. Capture these preferences now so every report-generating skill can apply them.

Preferences live in `kb.yaml` under `report_preferences:`. Factory defaults are in `plugins/kb/references/report-style-guide.md` — read that file first to see the field list, defaults, and variants.

Ask the user these 5 open-ended questions (AskUserQuestion, one at a time):

1. **"Who's the primary reader for reports from this vault? (e.g., just you, your team, or external readers like a blog or paper)"** → shapes the `audience` field.
2. **"Any style preferences — formal / plain / casual, short / long, anything else?"** → shapes the `register` field.
3. **"When covering technical projects, what balance of depth vs breadth do you want? (fewer items with deep walkthroughs, or broad surveys with short entries)"** → shapes the `depth` field.
4. **"How should I handle code in reports? Inline snippets, links to github, or walkthroughs with your editor open?"** → shapes the `code_handling` field.
5. **"Anything else worth noting upfront — diagram preferences, citation strictness, or any peeves to avoid?"** → appended verbatim to the `notes` field, plus used to tweak `diagrams` / `citations` if a clear signal appears.

For each answer:

- Open `plugins/kb/references/report-style-guide.md`.
- Pick the variant best matching the user's answer (`### Default`, `### Variant (formal)`, etc.).
- If the user's phrasing adds a specific constraint the variant doesn't cover, weave it in.
- Write the resulting free-text string to the corresponding field in `kb.yaml`.

If the user says "use defaults" / "I don't know" / skips a question — copy the `### Default` variant for that field unchanged.

The three fields not asked about at init (`self_containment`, `citations`, `argument_iteration`) get their `### Default` variants copied in unchanged. Tell the user these defaults are in place and can be updated later via `/kb-preferences update <field>`.

At the end of this section, briefly summarize the captured preferences back to the user and remind them:

> These preferences are stored in `kb.yaml` under `report_preferences:`. Skills will propose updates after reports based on your feedback — you'll approve before anything changes. You can edit them anytime via `/kb-preferences`.

### 6. Generate `kb.yaml`

Write `kb.yaml` at project root with paths, `output_formats`, obsidian config, and an `integrations` section. Example:

```yaml
project:
  name: "My Knowledge Base"
  description: "Knowledge base for [topic]"

paths:
  raw: raw/
  wiki: wiki/
  output: output/
  raw_articles: raw/articles/
  raw_papers: raw/papers/
  raw_repos: raw/repos/
  raw_notes: raw/notes/
  raw_images: raw/images/
  raw_transcripts: raw/transcripts/
  raw_datasets: raw/datasets/

source_strategy: mixed  # self-serve, assisted, or mixed

# Auto-save web pages fetched during any workflow (compile, query, evolve)
# Only saves pages that actually contributed to the output
auto_capture_web: true

output_formats:
  - markdown
  # - charts
  # - html
  # - marp

# How reports, wiki compilations, and query answers should be written.
# Populated from the 5 init questions + factory defaults in
# plugins/kb/references/report-style-guide.md. Free-text fields — skills
# read them as instructions, not parse them as keywords. Updatable via
# /kb-preferences or by hand-editing.
report_preferences:
  audience: |
    <<FILL: style-guide "audience" variant matching user's Q1 answer, tweaked with their phrasing>>
  register: |
    <<FILL: style-guide "register" variant matching user's Q2 answer>>
  depth: |
    <<FILL: style-guide "depth" variant matching user's Q3 answer>>
  code_handling: |
    <<FILL: style-guide "code_handling" variant matching user's Q4 answer>>
  diagrams: |
    <<FILL: style-guide "diagrams" default, unless Q5 signaled otherwise>>
  self_containment: |
    <<FILL: style-guide "self_containment" default>>
  citations: |
    <<FILL: style-guide "citations" default; use "external" variant if audience is publication>>
  argument_iteration: |
    <<FILL: style-guide "argument_iteration" default>>
  notes: |
    <<FILL: Q5 "anything else" answer verbatim, or leave empty if none>>

maintenance:
  manual: true
  scheduled: false

obsidian:
  vault_path: /path/to/vault
  recommended_plugins:
    - Web Clipper
    - Dataview
    - Obsidian Git

integrations:
  smaug:
    path: null  # set automatically when Smaug is installed

  # Optional tool availability (auto-detected during init)
  tools:
    whisper: false
    ffmpeg: false
    yt_dlp: false
    python_pptx: false
    pandas: false

# Entity types recognized by the system
entity_types:
  - concept
  - method
  - model
  - tool
  - benchmark
  - dataset
  - person
  - phenomenon
  - moc

# Maturity model
maturity:
  seedling_max_age_days: 30  # lint warns after this
  budding_min_words: 300
  budding_min_sources: 2
  evergreen_min_words: 800
  evergreen_min_sources: 3

# MOC auto-generation threshold
moc:
  min_articles_per_category: 5
```

When the user sets up Smaug (during init or later), save the install path here. The kb skill reads this to find Smaug without searching every time.

### 7. Scaffold Directories

Create:
- `raw/articles/`, `raw/papers/`, `raw/repos/`, `raw/notes/`, `raw/images/`, `raw/transcripts/`, `raw/datasets/`
- `wiki/`
- `output/`
- `.meta/` — **dotfolder** holding index/meta files. Must start with `.` — Obsidian ignores all dot-prefixed files and folders the same way it ignores `.obsidian/` itself, so meta files do not appear as graph nodes, search hits, or quick-switcher entries. Renaming this to `meta/` or `_meta/` would cause every meta file to show up as a huge hub in the graph.
- `.obsidian/` if creating a new vault

### 8. Initialize Meta Files

Meta files live in `.meta/` (NOT `wiki/`). They are read/written by the kb skill but invisible to Obsidian.

**`.meta/_index.md`:**
```markdown
---
title: "Wiki Index"
updated: YYYY-MM-DD
---

# Wiki Index

Master index of all wiki articles with source hashes for incremental compilation.

<!-- Categories will be added automatically as articles are compiled -->
```

**`.meta/_sources.md`:**
```markdown
---
title: "Source Mapping"
updated: YYYY-MM-DD
---

# Source Mapping

Mapping from raw sources to wiki articles they contributed to.

| Source | Type | Wiki Articles |
|--------|------|---------------|
<!-- Entries added automatically during compile -->
```

**`.meta/_categories.md`:**
```markdown
---
title: "Category Tree"
updated: YYYY-MM-DD
---

# Category Tree

Auto-maintained category hierarchy.

<!-- Categories created automatically as articles are compiled -->
```

**`.meta/_evolution.md`:**
```markdown
---
title: "Evolution Log"
updated: YYYY-MM-DD
---

# Evolution Log

Append-only log of auto-evolve actions.

| Date | Trigger | Action | Articles Affected |
|------|---------|--------|-------------------|
<!-- Entries added automatically by auto-evolve -->
```

**`.meta/_preference_history.md`:**
```markdown
---
title: "Preference Change History"
updated: YYYY-MM-DD
---

# Preference Change History

Append-only log of changes to `report_preferences` in `kb.yaml`.

| Date | Field | Trigger | Old → New |
|------|-------|---------|-----------|
<!-- Entries added by kb-preferences or reflection steps -->
```

**`.meta/_tags.md`:**
```markdown
---
title: "Tag Vocabulary"
updated: YYYY-MM-DD
---

# Tag Vocabulary

Canonical tag vocabulary for the wiki. All article tags must come from this list. See the kb skill's Tag Hygiene section for the namespacing rules.

## domain/* — broad subject area

<!-- Populated as articles are compiled -->

## method-family/* — class of technique

<!-- Populated as articles are compiled -->

## status/* — maturity mirror

- `status/seedling`
- `status/budding`
- `status/evergreen`

## Other

- `moc` — Map of Content pages
```

### 9. Write Project Files

- `CLAUDE.md` -- project instructions for future sessions
- `README.md` -- repo docs with prerequisites, setup, workflows, directory structure, and attribution for research skills

### 10. Next Steps Guidance

Tell user what to do next: add sources, compile, and list available workflows (`compile`, `query`, `lint`, `evolve`, `reflect`).

Mention omni-document support: "You can add PDFs, images, videos, audio, slides, code repos, datasets, notebooks, tweets, YouTube links, or any web URL to the raw/ directory. The compile workflow will automatically detect the type and extract knowledge."

## Common Mistakes

- Do not overwrite an existing `kb.yaml` without confirming with the user first.
- Do not skip the source-gathering strategy question -- it determines the entire workflow.
- Do not create `.obsidian/` inside a directory that is already an Obsidian vault.
- Do not assume output formats -- always ask the user.
- Do not skip creating `.meta/_evolution.md` -- it's needed for auto-evolve logging.
- Do not skip creating `.meta/_preference_history.md` -- it's needed for the reflection loop to log preference changes.
- Do not skip the Report Preferences section (§5.5). Without a populated `report_preferences:` block, downstream skills fall back to silent defaults and the user has no way to steer output style.
- Do not paste the field stub comments (`# picked from ...`) as literal values into `kb.yaml` -- replace each stub with the actual free-text preference string from the style-guide variant.
- **Do not place meta files under `wiki/`.** They belong in `.meta/` at the vault root. Files inside `wiki/` are indexed by Obsidian and will become graph nodes. The `.meta/` dotfolder is the structural guarantee that meta files stay out of the graph, search, and file explorer regardless of user settings.
- **Do not rename `.meta/` to `meta/` or `_meta/`.** The leading dot is load-bearing — that's what makes Obsidian skip the folder.
- **If migrating an existing vault** that has meta files under `wiki/` (`wiki/_index.md` etc.), move them to `.meta/` with the same filenames as part of init or the next compile.
