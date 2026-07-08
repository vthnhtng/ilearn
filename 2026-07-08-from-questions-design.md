# ilearn — From-Questions: Generate Roadmap from Interview Questions List

**Date:** 2026-07-08
**Status:** Design Specification
**Authors:** TungVT

## Overview

Add a new feature to ilearn that reads a `.txt` or `.md` file of interview questions, automatically groups them into categories and concepts, then scaffolds a full workspace (ROADMAP.md + concept folders with pre-populated questions). Available both as a standalone command and as an option during `/ilearn:init`.

## User Flow

1. User provides a questions file path (text or markdown)
2. The questions-classifier agent reads and groups the questions into categories → concepts
3. User reviews the classified roadmap, can add/remove/reorder
4. On approval → workspace is scaffolded with questions already split into each concept's `questions.md`

## Design Decisions (Approach 1)

- **New skill** `from-questions.md` → command `/ilearn:from-questions`
- **New agent** `questions-classifier.md` → reads raw questions, groups into categories, outputs YAML
- **Modify** `init.md` → add "From Questions File" as 3rd roadmap production option (alongside auto/manual)
- Reuses init.md's existing scaffold logic for folder/file creation

## New Files

| File | Purpose |
|------|---------|
| `commands/from-questions.md` | Command definition for `/ilearn:from-questions` |
| `.claude/skills/from-questions.md` | Skill file that orchestrates the flow |
| `.claude/plugins/ilearn/agents/questions-classifier.md` | Sub-agent for classification |

## Modified Files

| File | Change |
|------|--------|
| `plugin.json` | Add new skill and agent files |
| `.claude/skills/init.md` | Add "From Questions File" as 3rd roadmap option |

## Agent Output Format

```yaml
categories:
  - name: "Java Core"
    slug: "java-core"
    concepts:
      - name: "OOP Concepts"
        slug: "oop-concepts"
        difficulty: "beginner"
        estimated_hours: 3
        questions:
          - "Explain polymorphism with examples"
          - "What is the difference between abstraction and encapsulation?"
      - name: "Collections"
        slug: "collections"
        difficulty: "beginner"
        estimated_hours: 4
        questions:
          - "How does HashMap work internally?"
```

Same format as `roadmap-researcher.md` but with an optional `questions: [...]` per concept. Questions not assigned to any concept go into a "General" category.

## Skill Flow (from-questions.md)

1. Accept path arg (or ask user for it)
2. Validate file exists, read content
3. Spawn questions-classifier sub-agent
4. Present classified roadmap to user for approval
5. Handle edit requests (add/remove/reorder)
6. On approval → scaffold workspace
7. Unlike auto mode: each `questions.md` is pre-populated with classified questions

## Scaffold Reuse

Uses the same scaffold pattern from init.md:
- ROADMAP.md with numbered nested wikilinks (`[[concepts/<NN-cat>/<NN-con>/]]`)
- Concept subfolders with `theory.md`, `questions.md` (pre-populated), `answers.md`, `review.md`
- `interviews/` directory
- `.ilearn/config.json`
- Git init + commit

## Edge Cases

- **File not found** → ask for valid path or offer paste fallback
- **Empty file / no parseable questions** → error with explanation
- **Too many questions** (>100) → warn, still proceed
- **Too few questions** (<3) → warn it may not produce a meaningful roadmap
- **Already a workspace** (ROADMAP.md exists) → warn + confirm
- **Path is a directory** → scan for .txt/.md inside, let user pick
- **Malformed YAML from agent** → show raw output, ask user to help format
- **User cancels mid-way** → clean up partial files
