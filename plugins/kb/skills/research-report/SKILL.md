---
user-invocable: true
description: Summarize deep research results into a markdown report, covering all fields, skipping uncertain values.
allowed-tools: Read, Write, Edit, Glob, Bash, AskUserQuestion
---
> **Attribution:** Originally authored by [Weizhena](https://github.com/Weizhena/Deep-Research-skills). Included with attribution for use in the Deep query workflow.

# Research Report - Summary Report

## Trigger Method
`/research-report`

## Execution Flow

### Step 0: Load Report Preferences

Read the vault's `kb.yaml` and extract the `report_preferences:` block. These are free-text prose instructions written by the user at init (`kb-init` §5.5) or via `/kb-preferences`. They control audience, register, depth, code handling, diagrams, self-containment, citations, and argument iteration for this report.

**If the block is present:** apply every field as an instruction to the prose you write in subsequent steps. Treat the field text literally — it is the working rule, not a keyword list. The Python generator script written in Step 3 should also respect these preferences (e.g., if `depth` asks for multi-paragraph per-item walkthroughs, emit the item template with multi-paragraph slots; if `diagrams` says ASCII-only, don't emit mermaid).

**If the block is missing:** fall back to factory defaults in `plugins/kb/references/report-style-guide.md` silently and include this line in the final report output:

> No `report_preferences` set in `kb.yaml` — using factory defaults. Run `/kb-preferences init` to customize.

**If `kb.yaml` itself is missing:** tell the user to run `kb-init` and stop.

**Per-task overrides.** If the user's current request explicitly contradicts a stored preference ("make this one short", "skip the diagrams for this report"), follow the request for this report only. Do NOT modify `kb.yaml` — that's what the reflection step is for.

### Step 1: Locate Results Directory
Find `*/outline.yaml` in current working directory, read topic and output_dir configuration.

### Step 2: Scan Optional Summary Fields
Read all JSON results and extract fields suitable for display in the table of contents (numeric, short indicators), such as:
- github_stars
- google_scholar_cites
- swe_bench_score
- user_scale
- valuation
- release_date

Use AskUserQuestion to ask the user:
- Which fields should be displayed in the table of contents besides item names?
- Provide dynamic option list (based on fields actually present in JSONs)

### Step 3: Generate Python Conversion Script
Generate `generate_report.py` in the `{topic}/` directory with requirements:
- Read all JSONs from output_dir
- Read fields.yaml to get field structure
- Cover all field values from each JSON
- Skip fields containing [uncertain] values
- Skip fields listed in uncertain array
- Generate markdown report format: table of contents (with anchor jumps + user-selected summary fields) + detailed content (organized by field categories)
- Save to `{topic}/report.md`

**Table of Contents Format Requirements**:
- Must include every item
- Each item displays: sequence number, name (anchor link), user-selected summary fields
- Example: `1. [GitHub Copilot](#github-copilot) - Stars: 10k | Score: 85%`

#### Script Technical Points (Must Follow)

**1. JSON Structure Compatibility**
Support two JSON structures:
- Flat structure: fields directly at top level `{"name": "xxx", "release_date": "xxx"}`
- Nested structure: fields in category sub-dicts `{"basic_info": {"name": "xxx"}, "technical_features": {...}}`

Field lookup order: top level -> category mapping key -> traverse all nested dicts

**2. Category Multilingual Mapping**
Category names in fields.yaml and JSON keys can be any combination (Chinese-Chinese, Chinese-English, English-Chinese, English-English). Must establish bidirectional mapping:
```python
CATEGORY_MAPPING = {
    "Basic Info": ["basic_info", "基本信息"],
    "Technical Features": ["technical_features", "technical_characteristics", "技术特性"],
    "Performance Metrics": ["performance_metrics", "performance", "性能指标"],
    "Milestone Significance": ["milestone_significance", "milestones", "里程碑意义"],
    "Business Info": ["business_info", "commercial_info", "商业信息"],
    "Competition & Ecosystem": ["competition_ecosystem", "competition", "竞争与生态"],
    "History": ["history", "历史沿革"],
    "Market Positioning": ["market_positioning", "market", "市场定位"],
}
```

**3. Complex Value Formatting**
- list of dicts (like key_events, funding_history): format each dict as one line, separate kv with ` | `
- Regular lists: short lists joined with commas, long lists displayed with line breaks
- Nested dicts: recursively format, display with semicolons or line breaks
- Long text strings (over 100 characters): add line breaks `<br>` or use blockquote format for better readability

**4. Additional Field Collection**
Collect fields present in JSON but not defined in fields.yaml, put into "Other Information" category. Note filtering:
- Internal fields: `_source_file`, `uncertain`
- Nested structure top-level keys: `basic_info`, `technical_features`, etc.
- `uncertain` array: display each field name line by line, don't compress into one line

**5. Uncertain Value Skipping**
Skip conditions:
- Field value contains `[uncertain]` string
- Field name is in `uncertain` array
- Field value is None or empty string

### Step 4: Execute Script
Run `python {topic}/generate_report.py`

### Step 5: Preference Reflection

After `report.md` is written, scan the conversation for preference signals — moments where the user corrected your approach, accepted a non-obvious style choice silently, or said "remember this" / "for every work I do, remember X".

Follow the **Reflection Protocol** defined in `plugins/kb/skills/kb-preferences/SKILL.md`:

1. Scan for signals tied to specific `report_preferences` fields.
2. If signals found, propose concrete edits (show old-value excerpt → new-value excerpt).
3. Present proposals as a numbered list. Ask the user: `[a] all / [n] none / [e] edit each / [1,3] subset`.
4. On approval, write updates to `kb.yaml` and append entries to `.meta/_preference_history.md` with trigger `reflection after /research-report`.
5. If no signals found, skip silently.

**Signal strength rules** (from kb-preferences):
- Don't propose a change for a single offhand remark. Require explicit "remember this" phrasing, two occurrences, or feedback directly tied to this report.
- Don't propose contradictory changes. If the current field already captures the feedback, skip.
- Keep field text under ~400 words total. If a field would bloat past that, start a `notes` bullet instead.

## Output
- `{topic}/generate_report.py` - Conversion script
- `{topic}/report.md` - Summary report (styled per `report_preferences:` in `kb.yaml`)
- Optionally: updates to `kb.yaml` + one or more appended entries in `.meta/_preference_history.md` if the reflection step produced approved changes