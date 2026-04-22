# Report Style Guide — Factory Defaults

Source of factory-default text for `report_preferences` fields in a vault's `kb.yaml`.

**Read by:** `kb-init` (at vault creation) and `kb-preferences` (at reset time).

**Not read at report-generation time.** Once a vault is initialized, `kb.yaml`'s `report_preferences:` block is authoritative. Skills (`research-report`, `kb compile`, `kb query`) load preferences from `kb.yaml`, not this file.

**Format:** one top-level section per preference field. Each section has a `### Default` subsection (used when the user accepts defaults) and zero or more `### Variant (...)` subsections that the init skill can pick from based on user answers.

**How init uses this file:** the init skill reads the user's free-text answers to the 5 onboarding questions, picks the best-matching variant per field, optionally adjusts wording with the user's specific phrasing, and writes the result into `kb.yaml` as a free-text string. The resulting field is the working rule — it does not reference this file afterward.

---

## audience

Who the report is written for. Shapes framing, polish level, and hedging.

### Default (for own learning)

For my own learning and reference. Skip polish. Favor depth over summary. Honest framing about what's documented vs my own interpretation — say "I think" when I'm interpreting, "the paper says" when I'm citing. Flag `[uncertain]` claims explicitly, don't hide them behind confident prose. No need for executive-summary wrappers or "in this report we will" throat-clearing. Assume the reader (me) is willing to read carefully.

### Variant (for team audience)

For a small internal team. Assume shared context on broad topic but not on specific technical details. Light polish. Keep opinion sections labeled. Include a short TL;DR at the top. Technical depth OK but glossaries for terms specific to this investigation.

### Variant (for external / publication)

For external readers (blog post, paper, public share). Polished prose. Strong opening. Clear thesis. Every claim citeable. Remove "I think" hedges in favor of explicit confidence markers. No raw `[uncertain]` flags — either verify the claim or drop it. Structured like a paper: introduction, body sections, discussion, limitations.

---

## register

Tone and sentence-level style.

### Default (plain)

Plain, direct language. Short sentences — split multi-idea sentences on an em-dash or semicolon rather than stacking clauses with "and", "which", "that". Drop academic connectives (`notably`, `crucially`, `indeed`, `it should be noted`, `moreover`, `furthermore`). Active voice. First person is allowed and preferred over hedges like "one might observe". Contractions are fine. Paragraphs run 3–5 sentences, not 10-sentence walls. Prefer plain words over Latinate alternatives: `use` not `utilize`, `show` not `demonstrate`, `because` not `owing to the fact that`, `help` not `facilitate`, `about` not `with respect to`, `part` not `component` (usually), `thing` when it's actually just a thing.

**No scaffolding.** Drop sentences that announce what the next section will do. "This subsection explains X", "before we discuss Y, let's set the scene", "the four subsections below follow the same template: 1. ... 2. ..." — all scaffolding. The reader discovers structure by reading. Drop forward-reference promises too ("we'll come back to this", "more on this below"); just come back to it when you do.

**No production-process narration.** Describe what was surveyed and which sources were consulted, not who/what did the surveying or what pipeline produced the output. "We reviewed the README, DeepWiki pages, and arXiv paper per system" is fine; "N parallel sub-agents made X tool calls", "drawn from per-item JSONs with 100% validator coverage", "each sub-agent covered 2 items" are not. Pipeline artifacts are irrelevant to a reader trying to understand findings.

**No self-evaluation.** Don't grade your own analysis inside the analysis. "The report's sharpest original analysis", "this is the most important insight", "the key takeaway here" are self-grading. Present the analysis; let the reader judge what's sharp or important. You can label a section "Where this might be wrong" — that's flagging a weakness, not grading a strength.

### Variant (formal)

Formal, polished prose suitable for a paper or white paper. Full sentences, no contractions. Third person or impersonal constructions OK. Academic connectives are fine in moderation — but still don't overuse them. Defined technical vocabulary expected; glossaries optional. Paragraph length flexible.

### Variant (casual)

Conversational tutorial tone. Second person ("you'll notice"). Contractions throughout. Short punchy sentences. Humor and asides allowed. Occasional lists where prose would be tedious.

---

## depth

How much content per item and how much cross-cutting synthesis.

### Default (deep walkthroughs)

Fewer items covered more thoroughly. For a survey of N items, each item gets a multi-paragraph profile: who built it, when, what it is in one sentence, what problem it solves, its interface (with at least one concrete example), an analogy to something the reader likely knows, and how it relates to the other items. Appendix-style dossiers for the full set when the survey is large. Cross-cutting synthesis sections that compare items across axes (patterns, taxonomies, timelines) — these are the value-add over just listing items.

### Variant (survey)

Broad coverage. Short per-item sections (1–2 paragraphs max). Emphasis on completeness of the set, not depth per item. Comparison tables do most of the work. Cross-cutting analysis is shorter — one section, not multiple.

### Variant (overview)

Top-level map only. Named items, one-line description each, grouped. Cross-cutting analysis becomes the main content. Use when the goal is to orient the reader, not to teach them details.

---

## code_handling

How to incorporate code into reports. Scales with codebase count and snippet length.

### Default (mixed — scales by codebase count and snippet size)

**Small snippets (under ~30 lines):** inline in a code block, immediately followed by a prose walkthrough naming each moving part. The reader should not need to open an editor for these.

**Larger snippets or full files:** link to the source URL (github, local path, or wherever the code actually lives). A short prose explanation in the report. Don't reproduce entire files inline.

**Deep walkthroughs of 1–2 codebases:** this is the case where asking the reader to open their editor is reasonable. Tell them which files to open, then guide line-by-line with file:line references. The report acts as narration.

**3+ codebases:** switch to link-heavy mode. For each codebase, link to the relevant file/function on github (or a permalink), pull out the 5–10 key lines inline if they're worth dissecting, and keep prose lean. The reader uses links to context-shift between codebases.

**Never reproduce entire codebases inline.** If you find yourself writing more than ~100 lines of quoted code in one section, something has gone wrong.

### Variant (inline-walkthrough)

All code inline regardless of length, with aggressive annotation. Report is self-contained — reader never has to leave the document. Suitable for tutorials, not surveys.

### Variant (link-external)

Minimal inline code. Every code reference is a link to the source. Report is short and dense; reader does the navigation.

---

## diagrams

Visual aids — what kind, when to use them.

### Default (ascii + tables)

ASCII diagrams for flows, pipelines, and block-level relationships. Markdown tables for comparing items across axes. Mermaid only when specifically requested — ASCII renders reliably everywhere, mermaid does not. Drop diagrams when prose is equally clear; don't add them as decoration. For hierarchies and taxonomies, tables almost always beat tree diagrams.

### Variant (mermaid)

Mermaid diagrams preferred for flows and state machines. Tables still used for multi-axis comparisons. Reader is on a platform that renders mermaid (Obsidian, GitHub, Notion).

### Variant (prose-only)

No diagrams. All relationships expressed in prose, optionally supported by tables. Use when the target environment does not render diagrams reliably, or when the reader prefers reading over viewing.

---

## self_containment

How much context the report carries inside itself vs deferring to external sources.

### Default

Every term is glossed the first time it appears. Every project, model, paper, or tool mentioned gets a mini-profile the first time it appears: who built it (org / team), what it is in one sentence, when it came out. No `[see appendix]` deferrals in the main narrative — either inline the context or don't mention the thing yet. Interface examples (API calls, config files, CLI invocations) shown with enough surrounding context that the reader can parse them without guessing. Bare code drops without context are not acceptable.

Applies to depth-default reports, not survey/overview reports (where mini-profiles would bloat the prose).

### Variant (loose)

Assume shared background on the topic. Introduce only the items that are novel to this specific report. Use when writing follow-ups or for readers who asked for this report in the context of a larger investigation.

---

## citations

How to handle references, attributions, and primary sources.

### Default (ground in published literature)

Ground every taxonomic and historical claim in published literature. Cite with arXiv ID, venue (or "preprint" if unpublished), and publication date. Format: `Author et al. "Title." arXiv:2509.02547 (TMLR, Jan 2026)`. Distinguish three kinds of claims:

- **Inherited:** directly taken from a cited source. Cite it.
- **Extended:** built on a cited source but adds new analysis. Cite the source, then mark what's new.
- **Original:** claim that the report is making for the first time. Say so explicitly, and note it's not yet peer-reviewed.

Save every cited PDF or blog to `raw/papers/` (or `raw/articles/`) so the sources live with the vault. When a cited source disagrees with current code or current project state, flag the disagreement explicitly — don't pretend it doesn't exist.

### Variant (for-self loose references)

Loose references are fine. Link to sources when convenient. No strict format. Save obviously-important sources to `raw/` but don't make a production of it.

---

## argument_iteration

How to structure and validate non-trivial arguments (justifying a design, defending a taxonomy, etc.).

### Default

For any non-trivial argument (more than a paragraph, making a claim that could reasonably be challenged):

1. **Draft** the argument with explicit premises and conclusion.
2. **Advisor pass.** Call advisor() for a second opinion on the argument's structure and assumptions.
3. **Verifier pass.** Dispatch a custom subagent briefed on argument analysis: identify hidden premises, find the strongest counter-argument, stress-test the conclusion. Not a code reviewer — an argument reviewer.
4. **Revise** with a changelog noting what was weakened, strengthened, or dropped in response to the verifier's critique.
5. **Engage the strongest counter-arguments, not straw versions.** If the verifier names a counter-argument, it gets its own subsection, not a dismissive footnote.
6. **Every argument section ends with "Where this might be wrong"** — a real list of genuine threats to the argument, not decorative humility. This section is one of the most important parts of any report.

Skip this iteration for arguments that are just restating uncontroversial facts or summarizing a single source.

### Variant (loose)

One draft, one revision. No verifier loop. Use for quick memos or when the argument is genuinely uncontroversial.

---

## Applying this file

`kb-init` reads this file during vault setup, asks the user 5 open-ended questions about how they want reports written, picks the best-matching `### Default` or `### Variant` for each preference field, and writes the result into the vault's `kb.yaml` under `report_preferences:`. The user's specific phrasing may be woven into the default text.

After init, the vault's `kb.yaml` is the authoritative source. This file is consulted again only during:

- `/kb-preferences reset <field>` — copies the factory default for one field back into kb.yaml
- `/kb-preferences reset-all` — full regeneration from this file + re-asked questions
- `/kb-preferences init` — re-runs the init flow

Skills never read this file at report-generation time.
