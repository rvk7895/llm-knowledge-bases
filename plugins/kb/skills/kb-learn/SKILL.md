---
name: kb-learn
description: "Use when the user wants to study or test their understanding of knowledge base content, generate review questions, apply the Feynman technique, do active recall, or when they say 'quiz me', 'test my knowledge', 'help me learn', 'Feynman this', 'explain it back', or 'study mode'."
---

## Overview

Turns passive wiki reading into active learning. Generates study questions, Feynman challenges, and gap analysis from wiki articles. The user explains concepts back in plain language; Claude identifies where the explanation breaks down and what's missing.

**First action:** read `kb.yaml` for paths. If missing, tell the user to run `kb-init` and stop.

## Learning Modes

Ask the user which mode they want, or infer from context:

| Mode | What it does |
|------|--------------|
| **Quiz** | Generate factual recall questions from one or more articles |
| **Feynman** | User explains a concept in plain language; Claude diagnoses gaps |
| **Deep dive** | Socratic dialogue -- Claude asks probing questions to surface understanding depth |
| **Scan** | Quick coverage check -- which topics in the wiki does the user know well vs. not at all |

## Model Strategy

| Task | Executor |
|------|----------|
| Read and extract key claims from articles | haiku subagents |
| Generate questions and Feynman prompts | sonnet |
| Evaluate user explanations and diagnose gaps | opus |

## Workflow

### Step 1: Choose Scope

Ask or infer:
- **Single article** -- deep study of one topic
- **Category** -- study all articles in a folder
- **Whole wiki** -- broad coverage scan
- **User-specified topic** -- Claude finds the relevant articles

### Step 2: Extract Key Claims (haiku subagents)

For each article in scope, dispatch a **haiku** subagent to extract:
- Core claims (the 3-5 most important facts or ideas)
- Definitions (terms the article introduces or explains)
- Relationships (how this concept connects to others in the wiki)
- Common misconceptions (if the article flags any)

Return structured extraction to sonnet.

### Step 3: Generate Study Material (sonnet)

Based on the chosen mode:

**Quiz mode** -- generate 5-10 questions per article:
- Mix question types: factual recall, "why does X happen", "compare A and B", "what would happen if..."
- Order by difficulty: start easy, get harder
- Include the article source for each question (for self-checking)
- Do NOT include answers inline -- present them separately after the user attempts

**Feynman mode** -- generate a single challenge prompt:
```
Explain [[Article Title]] as if you're teaching it to someone who has never heard of it.
Don't use jargon from the article. Use an analogy if it helps.
Take your time. I'll ask follow-up questions when you're done.
```

**Deep dive mode** -- prepare a sequence of 5 Socratic questions that go from surface to depth:
1. What is X? (definition)
2. Why does X matter? (significance)
3. How does X work? (mechanism)
4. What breaks X or limits it? (edge cases)
5. How does X relate to Y? (connections to other wiki articles)

**Scan mode** -- list all topics in scope with a one-line summary and ask the user to rate their confidence (1-5) for each. Use ratings to prioritize which topics to study first.

### Step 4: Run the Session

Present the study material and wait for user responses. Keep the conversation interactive:

- In **Quiz** mode: present one question at a time, wait for answer, then reveal the correct answer with the source article
- In **Feynman** mode: after the user explains, opus evaluates the explanation against the source article
- In **Deep dive** mode: ask questions one at a time, follow up based on the answer
- In **Scan** mode: collect confidence ratings, then suggest a study order

### Step 5: Feynman Gap Analysis (opus)

In Feynman mode, after the user's explanation, opus compares it to the source article and diagnoses:

**What the user got right** -- concepts explained accurately (reinforce these)

**Gaps** -- important claims from the article that weren't mentioned:
- "You didn't mention X, which is important because..."

**Misconceptions** -- things the user said that contradict the article:
- "You said X, but the article says Y -- this matters because..."

**Oversimplifications** -- concepts explained correctly in spirit but missing critical nuance:
- "Your analogy works for the basic case but breaks down when..."

Present the gap analysis clearly. Don't be harsh -- frame gaps as "here's what to revisit", not "you were wrong."

### Step 6: Suggest Follow-Up

After the session, offer:
- "Want to revisit the articles where you had gaps?"
- "Want me to generate a summary of what you learned today and file it to `output/`?"
- "Want to schedule a review of the weak spots in a future session?"

## Output Format (if saving session)

If the user wants to save the session, write to `output/learn-YYYY-MM-DD-TOPIC.md`:

```markdown
---
title: "Learning Session — TOPIC — YYYY-MM-DD"
type: learn
mode: feynman | quiz | deep-dive | scan
articles: [list of articles covered]
date: YYYY-MM-DD
---

# Learning Session — TOPIC

## Articles Covered
- [[Article A]]
- [[Article B]]

## Gaps Identified
- X was not mentioned (source: [[Article A]])
- Y was oversimplified

## Strong Areas
- Z was explained accurately

## Next Review
Articles to revisit: [[Article A]], [[Article B]]
```

## Common Mistakes

- Giving answers before the user attempts -- always wait for their response first
- Overwhelming the user with all questions at once -- present one at a time in Quiz and Deep dive modes
- Being harsh in Feynman gap analysis -- frame everything as "here's what to revisit", keep it constructive
- Running Feynman mode on a topic with no wiki article -- the analysis only works when there's a source to compare against; if the article doesn't exist, suggest compiling it first
- Treating Scan mode as a quiz -- it's a self-assessment, not a test; trust the user's confidence ratings
