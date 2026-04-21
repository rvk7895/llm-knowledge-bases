---
name: kb-preferences
description: "Use when user wants to view, update, reset, or audit report preferences stored in vault kb.yaml. Triggers on '/kb-preferences', 'update report prefs', 'change report style', 'my preferences', or when a passive reflection step in another skill needs to propose preference updates."
user-invocable: true
allowed-tools: Read, Write, Edit, Glob, Bash, AskUserQuestion
---

## Overview

Manage the `report_preferences:` block in the vault's `kb.yaml`. This is a free-text preference store — each field is a prose instruction string that `research-report`, `kb compile`, and `kb query` read when generating output. Factory defaults come from `plugins/kb/references/report-style-guide.md`, which is consulted only at init / reset time.

**Update path in three places:**
1. User runs `/kb-preferences ...` explicitly — this skill.
2. User hand-edits `kb.yaml` — no skill involvement.
3. Another skill's reflection step proposes updates after finishing a report — that skill calls into this one.

**Every write confirms with the user first** and logs the change to `.meta/_preference_history.md`. The kb.yaml and the history file are the only two places state lives.

## Prerequisites

- `kb.yaml` exists at the vault root. If missing, tell the user to run `kb-init` and stop.
- `plugins/kb/references/report-style-guide.md` exists (plugin-shipped; should always be present).
- `.meta/_preference_history.md` exists. If missing, create it with the template below before any write.

## Fields managed

All fields under `kb.yaml` → `report_preferences:`:

| Field | What it controls |
|---|---|
| `audience` | Who the report is written for; framing, polish, hedging |
| `register` | Tone, sentence style, vocabulary |
| `depth` | Items-covered × per-item-depth tradeoff |
| `code_handling` | Inline vs linked vs walkthrough, scaled by codebase count |
| `diagrams` | ASCII / mermaid / prose / tables |
| `self_containment` | Glossing, mini-profiles, no appendix deferrals |
| `citations` | arXiv-style rigor vs loose references |
| `argument_iteration` | Draft → advisor → verifier loop + "where this might be wrong" |
| `notes` | Free-form addendum; grows via reflection |

## Modes

The skill detects mode from user invocation. Parse the arguments after `/kb-preferences`:

| Invocation | Mode | Action |
|---|---|---|
| `/kb-preferences` (bare) | interactive | Show current prefs, ask what to update |
| `/kb-preferences init` | init | Re-run the 5 init questions, regenerate all fields from style-guide + user answers |
| `/kb-preferences update <field>` | update-one | Focus on one field; re-ask matching question or accept free-text |
| `/kb-preferences add-note "..."` | add-note | Append the quoted string as a new line/paragraph under `notes:` |
| `/kb-preferences reflect` | reflect-now | Run the reflection step against the current conversation explicitly |
| `/kb-preferences review` | review | Print current `report_preferences:` block read-only |
| `/kb-preferences reset <field>` | reset-one | Copy factory default for one field; confirm first |
| `/kb-preferences reset-all` | reset-all | Full reset to defaults; confirm first |
| `/kb-preferences history` | history | Print `.meta/_preference_history.md` |

### Mode: interactive (bare invocation)

1. Read `kb.yaml` `report_preferences:` section.
2. If missing, tell user to run `/kb-preferences init` first and stop.
3. Print each field with a truncated preview (~80 chars, "..." if longer).
4. AskUserQuestion: "Which field would you like to update?" with the 9 field names + "none / cancel" as options.
5. If a field picked, delegate to update-one mode for that field.

### Mode: init

Same flow as `kb-init` §5.5. Ask the 5 open-ended questions (see kb-init SKILL.md for the exact phrasing):

1. Primary reader → `audience`
2. Style preferences → `register`
3. Depth vs breadth → `depth`
4. Code handling → `code_handling`
5. Anything else → `notes` (+ opportunistic tweaks to `diagrams` / `citations` if signaled)

For each answer, open `plugins/kb/references/report-style-guide.md`, pick the best-matching variant, optionally weave in user phrasing, write to the corresponding field in `kb.yaml`.

Fields not asked at init (`self_containment`, `citations`, `argument_iteration`) get their `### Default` variants unchanged unless question 5 signaled a change.

**Confirm the full assembled block with the user before writing.** Show the block, ask "Write this to kb.yaml? [y/n/edit]". On `edit`, let them hand-revise the text.

Log each changed field to `.meta/_preference_history.md`.

### Mode: update-one

1. Argument is the field name. Validate it's one of the 9 known fields. If `notes`, prefer add-note mode.
2. Show the current value of that field to the user.
3. AskUserQuestion the matching init question for that field.
4. Consult the style guide, pick a variant, weave in user phrasing.
5. Show the new proposed value, confirm with user ("Write? [y/n/edit]").
6. On confirmation, write to `kb.yaml` and log to history.

### Mode: add-note

1. Argument is a quoted string.
2. Read current `notes:` field.
3. Append the string as a new bullet (or paragraph if multi-sentence).
4. Confirm with user, write, log.

### Mode: reflect-now

Explicit reflection against the current conversation. Used when the user wants to force a pass.

1. Scan the conversation for preference signals (see Reflection Protocol below).
2. If signals found, propose updates with diffs.
3. Confirm, write, log.

If no signals, say so and exit.

### Mode: review

Read-only. Print the full `report_preferences:` block, each field shown in full (no truncation). No changes.

### Mode: reset-one

1. Argument is the field name.
2. Open `plugins/kb/references/report-style-guide.md`, pull the `### Default` variant for that field.
3. Show diff between current value and factory default.
4. AskUserQuestion: "Reset this field to factory default? [y/n]".
5. On yes, write to `kb.yaml`, log `reset-one` trigger in history.

### Mode: reset-all

1. Confirm loudly: "This will replace all 9 fields with factory defaults and re-run the 5 init questions. Continue? [y/n]".
2. On yes, behave like init mode.
3. Every field change is a separate history log entry with trigger `reset-all`.

### Mode: history

Print `.meta/_preference_history.md`. If missing, say "No history yet" and create the file per the template below.

## Reflection Protocol

Called in two situations:
- This skill's `reflect-now` mode (user explicit)
- The reflection step at the end of `research-report`, `kb compile`, or `kb query`

Algorithm:

1. **Scan the conversation for signals** across these categories:
   - **Corrections**: user pushed back on output ("no, make it plainer", "drop the academic connectives", "you're using too much jargon")
   - **Silent-accept signals**: user accepted a non-obvious style choice without comment → validation of that choice
   - **Explicit feedback**: user said "I liked X" or "next time, do Y"
   - **Meta-comments**: "for every work I do, remember X"

2. **Map each signal to a specific `report_preferences` field.**
   - Word-level complaints → `register`
   - Content coverage / depth complaints → `depth`
   - Code handling complaints → `code_handling`
   - Citation rigor requests → `citations`
   - Diagram preferences → `diagrams`
   - Self-containment feedback → `self_containment`
   - Argument validation requests → `argument_iteration`
   - Target-reader hints → `audience`
   - Anything else that's a one-off preference → `notes`

3. **Propose concrete edits, not abstract advice.** For each signal, show:
   ```
   Signal: "<short paraphrase of what user said or did>"
   Field: <field_name>
   Current (excerpt): "<~60 chars>..."
   Proposed (excerpt): "<~60 chars with the change>..."
   ```

4. **Present proposals as a numbered list** and ask:
   ```
   Apply these changes to kb.yaml?
     [a] all   [n] none   [e] edit each one   [1,3] just those
   ```

5. **On apply**: write accepted changes to `kb.yaml`, append entries to `.meta/_preference_history.md`.

6. **If no signals found**: skip silently unless mode is `reflect-now` (in which case say "no signals detected").

### Signal strength rules

- Don't propose a change for a single offhand remark. Require either (a) explicit "remember this" phrasing, (b) two occurrences of the same feedback, or (c) the feedback is directly tied to the report just produced.
- Don't propose contradictory changes. If the current field already captures the feedback, skip.
- Don't propose style-guide lists (e.g., don't bloat `register` with 40 forbidden words from one session). Keep field text under ~400 words; if a field is bloating, start a new `notes` bullet instead.

## `.meta/_preference_history.md` format

```markdown
---
title: "Preference Change History"
updated: YYYY-MM-DD
---

# Preference Change History

Append-only log of changes to `report_preferences` in `kb.yaml`.

| Date | Field | Trigger | Old → New |
|------|-------|---------|-----------|
| 2026-04-21 | register | user (/kb-preferences update) | "Plain..." → "Plain and very short..." |
| 2026-04-22 | notes | reflection after /research-report | (empty) → "Per-item profiles max ~150 words" |
```

Triggers to use in the `Trigger` column:

- `user (/kb-preferences <mode>)` — explicit user invocation
- `user (init)` — first-time setup during kb-init
- `reflection after <skill>` — reflection step from research-report / kb compile / kb query
- `reset-one` / `reset-all` — factory default restore
- `hand-edit detected` — if future tooling notices the user directly edited kb.yaml (not implemented now; reserved trigger)

For each write:
- Truncate old and new values to ~60 chars; use `...` if truncated.
- If a field previously did not exist, old value is `(unset)`. If a field is cleared, new value is `(empty)`.
- Update the `updated:` frontmatter to the current date.

## Common Mistakes

- Forgetting to confirm before writing. Every write path confirms. No silent updates except no-op exits when no signal is detected.
- Treating `notes` as a dumping ground. Apply the signal to the specific field if one fits; only fall back to `notes` if no field matches.
- Bloating fields indefinitely. Fields cap at ~400 words; past that, start a `notes` bullet instead.
- Reading `plugins/kb/references/report-style-guide.md` at report-generation time. That file is only consulted by this skill and `kb-init`, at init / reset time. Report-generating skills read `kb.yaml`.
- Renaming `kb.yaml` fields. The field names are a contract with the other skills (research-report, kb). Do not rename.
- Parsing the free-text field values. They're prose instructions for the next report, not keyword lists.
- Skipping history logging. Every approved change goes in `.meta/_preference_history.md`. If the file is missing, create it from the template.
