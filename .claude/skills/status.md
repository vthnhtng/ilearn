---
name: status
description: "Show progress overview — concepts reviewed, scores, weak areas, retry candidates, and suggested next steps."
---

# ilearn-status — Progress Overview

## When to use
User types `/ilearn-status` to see their overall interview prep progress.

## Prerequisites
- A workspace initialized with `/ilearn-init` — ROADMAP.md must exist in cwd.
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn-init` first to create one." and stop.

## Flow

### Step 1: Detect workspace

Check if `ROADMAP.md` exists in cwd. If not → warn and stop.

### Step 2: Read ROADMAP.md header

Parse the header block. Extract:
- Topic from `# Roadmap: <topic>`
- Current/target levels from `**Current Level:**` and `**Target Level:**`
- Concept counts from `**Concepts:** <completed>/<total> (<percentage>%)`
- Overall score from `**Overall Score:** <score>/10` (or `—` if none)
- Last updated date

### Step 3: Read all concept entries

1. Parse all lines matching `- [x]` or `- [ ]` patterns with `[[concepts/<slug>/]]`.
2. For each `[ ]` (unreviewed) concept, read `concepts/<slug>/review.md` if it exists — some may have `status: in_progress` from a low first attempt.
3. For each `[x]` concept, extract its score from the ROADMAP.md line.
4. Categorize each concept:

| Category | Condition |
|----------|-----------|
| **Reviewed** (pass) | `[x]` and score >= 7 |
| **Needs Retry** | `[x]` and score < 7, OR `[ ]` with review.md `status: in_progress` |
| **New** | `[ ]` and no review.md, or review.md `status: new` |

### Step 4: Read interview logs (optional)

1. List files under `interviews/` directory (if it exists).
2. For each `.md` file (excluding `.gitkeep`), extract:
   - Date from filename or content
   - Score and Result (PASS/FAIL)
   - Number of concepts covered
3. Keep the last 5 interview logs for display.

### Step 5: Build the report

Print a formatted report:

```
📊 ilearn Status — <Topic>

Level:       <current> → <target>
Progress:    <completed>/<total> concepts (<percentage>%)
Overall:     <score>/10
Last Study:  <last_reviewed_date or "—">

── Review Status ──

Reviewed (pass):   <N>
  ✅ <Concept Name> — <score>/10 (Reviewed: <date>)
  ✅ <Concept Name> — <score>/10 (Reviewed: <date>)

Needs Retry:      <N>
  🔄 <Concept Name> — <score>/10 — <reason>
  🔄 <Concept Name> — <score>/10 — <reason>

New (not started): <N>
  📄 <Concept Name>
  📄 <Concept Name>

── Weak Areas ──

Based on scores, these dimensions need the most work:
  Understanding:  avg <N>/10
  Depth:          avg <N>/10
  Communication:  avg <N>/10

<1-2 sentence analysis of the weakest dimension>

── Recent Interviews ──

<date> — <score>/10 — <PASS/FAIL> — <N> concepts
<date> — <score>/10 — <PASS/FAIL> — <N> concepts
<or "No interviews yet — run /ilearn-interview">

── Suggested Next Steps ──

1. Review: <concept name> (needs retry, score <N>/10)
   /ilearn-review concepts/<slug>/
2. Study: <concept name> (new, high priority for <target> level)
   /ilearn-theory concepts/<slug>/
3. Mock Interview: after reviewing more concepts
   /ilearn-interview
```

### Step 6: Computing Weak Areas

1. Read all `concepts/<slug>/review.md` files that have `status: reviewed` or `in_progress`.
2. Extract the breakdown (understanding, depth, communication) from each.
3. Average each dimension across all reviewed concepts.
4. Identify the lowest dimension — that's the weak area.
5. Write a short analysis based on which dimension is weakest:
   - **Understanding lowest**: "Focus on core concepts — your answers show gaps in fundamental understanding."
   - **Depth lowest**: "Go deeper — study internals, edge cases, trade-offs for each concept."
   - **Communication lowest**: "Practice structuring answers — use frameworks like STAR or problem-solution-impact."

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn-init` first.
- **No concepts at all**: Print "ROADMAP.md is empty. Add concepts to get started." and stop.
- **No reviewed concepts**: Print report with 0 reviewed, all concepts as New.
- **No interviews yet**: Show "No interviews yet — run /ilearn-interview" in that section.
- **Missing review.md files**: Treat unreviewed concepts as New with score 0.
- **review.md has no breakdown**: Default to all 0s for that concept (skip from weak areas avg).
- **Corrupt ROADMAP.md**: If header parsing fails, show "ROADMAP.md format error — check the file." and show raw content.
