# ilearn — AI-powered Technical Interview Training Workspace

**Date:** 2026-07-07
**Status:** Design Specification
**Authors:** TungVT

---

## 1. Overview

ilearn is a plugin system for AI coding agents (Claude Code, Codex, Cursor, etc.) that creates structured workspaces for technical interview preparation. Users define a target role/topic, and ilearn manages the full learning loop: roadmap generation → concept learning → Q&A review → mock interviews → progress tracking.

### Core Principle

All data lives in plain Markdown files in a single workspace folder — zero external storage, fully portable, compatible with Obsidian and any Markdown editor.

---

## 2. Workspace Format

### 2.1 Directory Structure

```
<workspace-root>/
├── README.md                  # Workspace overview
├── ROADMAP.md                 # Master checklist + progress
│
├── concepts/                  # One subfolder per concept
│   ├── java-oop/
│   │   ├── theory.md          # Learning notes
│   │   ├── questions.md       # Review questions
│   │   ├── answers.md         # User's answers
│   │   └── review.md          # AI review + YAML state
│   ├── spring-ioc/
│   └── ...
│
├── interviews/                # Mock interview logs
│   └── 2026-07-07_java-springboot_7.5.md
│
└── .ilearn/
    └── config.json            # Plugin config, agent-version, timestamps
```

### 2.2 ROADMAP.md Format

Single master file tracking all concepts and overall progress.

```markdown
# Roadmap: Java Spring Boot Backend Developer

**Current Level:** Junior → **Target Level:** Middle
**Concepts:** 4/32 (12.5%)
**Overall Score:** 6.8/10
**Last Updated:** 2026-07-07

---

## 1. Core Java Foundation

- [x] Java OOP | Score: 8/10 | [[concepts/java-oop/]] | Reviewed: 2026-07-07
- [ ] Collections Framework | [[concepts/java-collections/]]
- [ ] Generics | [[concepts/java-generics/]]

## 2. Spring Framework

- [ ] Spring IOC / DI | [[concepts/spring-ioc/]]
- [ ] Spring Boot Auto Configuration | [[concepts/spring-boot-autoconfig/]]
```

#### Rules

- `[x]` = concept reviewed and scored. `[ ]` = not yet reviewed.
- Lines with `[x]` include `| Score: N/10 |` and `| Reviewed: YYYY-MM-DD |`.
- `[[concept-folder-name/]]` is an Obsidian-compatible wikilink to the concept subfolder.
- Categories are `##` headings. Order implies priority sequence.

### 2.3 Concept Subfolder Format

Each concept has exactly four Markdown files:

**theory.md** — Free-form learning notes. Machine-readable only for context.

**questions.md** — One question per line or section:

```markdown
1. What is the difference between `==` and `.equals()` in Java?
2. Explain method overriding vs overloading.
```

**answers.md** — User's written answers, index-aligned with questions. Can be empty (user may answer via chat in ilearn-review instead).

```markdown
### Q1
== compares references, .equals() compares values...
```

**review.md** — Single source of truth for concept state. YAML frontmatter + markdown body:

```yaml
---
status: reviewed           # new | in_progress | reviewed
score: 7
breakdown:
  understanding: 8
  depth: 6
  communication: 7
attempt: 1
reviewed_at: 2026-07-07
---

## Feedback

Good understanding of core concepts, but needs more depth on edge cases.
```

### 2.4 Interview Log Format

File: `interviews/<date>_<topic>_<overall-score>.md`

```markdown
# Interview Log

**Date:** 2026-07-07
**Topic:** Java Spring Boot Backend
**Duration:** 30 min
**Concepts Covered:** [[concepts/java-oop/]], [[concepts/spring-ioc/]], [[concepts/spring-boot-autoconfig/]]
**Result:** PASS | Overall Score: 7.5/10

## Questions & Answers

### Q1: Explain Spring Bean lifecycle
**Answer:** ...
**Evaluation:** Good, missed postConstruct details. 7/10
**Follow-up:** What is the difference between Singleton and Prototype scope?

...

## Summary

**Strengths:** Spring fundamentals, dependency injection
**Weaknesses:** Transaction management, AOP concepts
**Final Verdict:** PASS — ready for Mid-level interviews
```

---

## 3. Skill Architecture

### 3.1 Plugin Directory Layout

```
├── plugin.json                 # Plugin manifest for distribution
├── marketplace.json            # Source registry for plugin install
│
.claude/
├── skills/
│   ├── ilearn-init.md         # Initialize workspace
│   ├── ilearn-review.md       # Review & score a concept
│   ├── ilearn-interview.md    # Mock interview session
│   ├── ilearn-theory.md       # Deep-dive concept teaching
│   ├── ilearn-status.md       # Progress overview
│   └── plugin-install.md      # Plugin installer
│
├── plugins/
│   └── ilearn/
│       ├── agents/
│       │   ├── roadmap-researcher.md
│       │   ├── reviewer.md
│       │   ├── interviewer.md
│       │   └── teacher.md
│       └── references/
│           └── workspace-format.md
```

### 3.2 Skill: `ilearn-init`

**Triggers:** `/ilearn:init` or `/ilearn:init "topic string"`

**Flow:**

1. **Greet & input**
   - If no topic arg → ask "What role/topic do you want to prepare for?"
   - Ask current level and target level (Fresh/Junior/Mid/Senior)
   - If auto roadmap → ask target timeline (1mo/3mo/6mo/any)

2. **Roadmap creation** (user chooses auto or manual)
   - **Auto**: spawn `roadmap-researcher` agent with prompt including topic, current-level, target-level, timeline. Agent researches deeply (web search, multiple sources) and returns a prioritized concept list with categories. Show to user for approval; user can edit/reorder.
   - **Manual**: create folder structure only. User provides their own ROADMAP.md.

3. **Scaffolding** — create:
   - `README.md` with topic + level summary
   - `ROADMAP.md` with parsed concept list
   - `concepts/<slug>/` for each concept, each containing `theory.md`, `questions.md`, `answers.md`, `review.md`
   - `interviews/` (empty)
   - `.ilearn/config.json`
   - Git init + initial commit

### 3.3 Skill: `ilearn-review`

**Triggers:** `/ilearn:review` or `/ilearn:review concepts/java-oop/`

**Flow:**

1. If no concept arg → list concepts with status `new` or `in_progress`, user picks one
2. Read `theory.md` and `questions.md` from concept folder
3. If `questions.md` has entries → use those; otherwise AI generates questions based on `theory.md`
4. Present questions one at a time (or in batch — user preference)
5. If user answers in chat → write to `answers.md`
6. Agent scores based on rubric:
   - **Understanding** (1-10): Did they grasp the core idea?
   - **Depth** (1-10): Can they go beyond surface-level?
   - **Communication** (1-10): Is the answer structured and clear?
   - **Overall**: weighted average (U×0.4 + D×0.4 + C×0.2)
7. Write `review.md` with score breakdown and qualitative feedback
8. Update `ROADMAP.md` — mark `[x]`, set score, update date
9. If score >= 7 → status = `reviewed`; else → `in_progress` (needs retry)

### 3.4 Skill: `ilearn-interview`

**Triggers:** `/ilearn:interview`

**Precondition:** Workspace must exist with `ROADMAP.md` containing at least 1 reviewed concept.

**Flow:**

1. Determine interview parameters:
   - Ask duration (15/30/45/60 min)
   - Or auto-calculate: 5 questions per 30 min
2. **Interview mode** — agent plays strict interviewer:
   - Does NOT reveal answers during interview
   - Asks questions covering reviewed concepts in ROADMAP.md
   - Asks follow-up questions based on answers
   - Mixes easy → medium → hard difficulty
3. **Scoring** — each answer gets per-question score (1-10)
4. **Debrief** — after all questions:
   - Summary per question with score
   - Strengths & weaknesses
   - Final verdict: PASS (avg >= 6.5) or FAIL
5. Write interview log to `interviews/<date>_<topic>_<score>.md`
6. Optionally update concept `review.md` files whose concepts were covered

### 3.5 Skill: `ilearn-theory` (Optional — Implemented)

**Triggers:** `/ilearn:theory` or `/ilearn:theory concepts/java-oop/`

Agent reads `theory.md` (and related web search) and delivers an interactive lesson — explains the concept, shows examples, can answer follow-up questions. Appends new insights to `theory.md` afterwards.

**Files:** `.claude/skills/ilearn:theory.md` + `.claude/plugins/ilearn/agents/teacher.md`

### 3.6 Skill: `ilearn-status` (Optional — Implemented)

**Triggers:** `/ilearn:status`

Reads ROADMAP.md and all `review.md` files, generates a text summary: progress %, weak areas, which concepts need retry, suggested next steps.

**File:** `.claude/skills/ilearn:status.md`

### 3.7 Skill: `plugin-install`

**Triggers:** `/plugin-install` or `/plugin-install <name>@<source>` or `/plugin-install <path>`

**Purpose:** The bootstrap installer for distributing ilearn (and any future plugin) as a reusable package. Resolves source from `marketplace.json`, validates `plugin.json` manifest, copies declared files into `.claude/`.

**Files:**
- `.claude/skills/plugin-install.md` — the installer skill
- `plugin.json` — plugin manifest (name, version, files list)
- `marketplace.json` — source registry (maps aliases like `local` to paths)

**Installation flow:**
1. First-time user manually copies `plugin-install.md` into their project's `.claude/skills/`.
2. User runs `/plugin-install ilearn@local` to install all 10 ilearn files.
3. The skill reads `marketplace.json` → resolves `local` → `.` → reads `plugin.json` → copies each file to `<cwd>/<filepath>`.

**Edge cases:** Missing source registry, unknown source, missing manifest, name mismatch, files already exist (prompts overwrite), missing source files (skip with warning).

---

## 4. Sub-Agent Specifications

### 4.1 roadmap-researcher

**When spawned by:** `ilearn-init` (auto roadmap mode)

**Instructions (condensed):**
- You are a curriculum designer.
- Given topic, current-level, target-level, timeline — produce a prioritized list of concepts.
- Research with web search for up-to-date interview requirements.
- Group concepts into categories (max 6-8 categories).
- Each concept must be a concrete, single topic (e.g., "Java OOP" not "Java").
- Order: foundational first, advanced later. Interview-high-frequency first.
- Output as YAML with category, concept name, slug, difficulty, estimated hours.

### 4.2 reviewer

**When spawned by:** `ilearn-review`

**Instructions (condensed):**
- You are a senior technical interviewer.
- Read `theory.md` to understand what the user studied.
- Generate questions that test understanding, not memorization.
- Score fairly: 5 = acceptable, 7 = good, 9 = excellent.
- Provide actionable feedback: what was good, what was missing, what to study next.
- Write output structured as YAML frontmatter + markdown body.

### 4.3 interviewer

**When spawned by:** `ilearn-interview`

**Instructions (condensed):**
- You are a strict but fair interviewer.
- Do NOT reveal answers during the interview.
- For each answer, decide an immediate per-question score.
- Ask follow-ups when answers are shallow.
- At end: aggregate scores, determine PASS/FAIL.

---

## 5. Shared Reference: `workspace-format.md`

This file lives at `.claude/plugins/ilearn/references/workspace-format.md` and contains the exact workspace directory tree, ROADMAP.md syntax rules, review.md frontmatter schema, and interview log format. Every sub-agent reads this file at startup so it knows how to read/write workspace files correctly. This is the **contract** — if the format changes, update this file first.

---

## 6. Edge Cases & Constraints

| Scenario | Handling |
|----------|----------|
| User has no workspace yet | `ilearn-review` and `ilearn-interview` detect missing ROADMAP.md and prompt `/ilearn:init` first |
| Concept folder missing ROADMAP.md entry | Review skill treats it as untracked — refuses to create orphans; user must update ROADMAP.md |
| Multiple review attempts | `attempt` counter in review.md increments. Old feedback archived in body. Re-review only if score < 7 |
| Empty theory.md | Reviewer agent falls back to topic name + web search for questions |
| Interview with zero reviewed concepts | Blocked — must review at least 1 concept first |
| Workspace git repo | Every skill auto-commits after significant writes (review, interview) |
| Concurrent agent runs | Not supported — serialized per workspace |
| Obsidian compatibility | Format uses plain MD + `[[wikilinks]]`. No Obsidian-exclusive features. Works in any MD editor |

---

## 7. Implementation Order

| Phase | Skills | Milestone |
|-------|--------|-----------|
| 1 | `ilearn-init` | Can scaffold a workspace |
| 2 | `ilearn-review` | Can review + score + update progress |
| 3 | `ilearn-interview` | Can run mock interviews + log |
| 4 | `ilearn-theory`, `ilearn-status` | Enhanced features (Phase complete) |
| 5 | `plugin-install`, `plugin.json`, `marketplace.json` | Plugin distribution (Phase complete) |

---

## 8. Non-Goals

- No web dashboard or GUI — terminal/MD-only
- No spaced repetition scheduler (could be added later)
- No multi-user / sharing features
- No IDE extension — relies on Claude Code and similar CLI agents
- No package manager dependency — all Markdown, no build
