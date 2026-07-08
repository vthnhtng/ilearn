---
name: create-question
description: "Generate 5-10 review questions from a concept's theory.md and write them to questions.md. Requires theory.md to be populated first."
---

# ilearn-create-question — Generate Review Questions

## When to use
User types `/ilearn:create-question [concepts/<slug>/]` to generate review questions from existing theory notes.

## Prerequisites
- A workspace initialized with `/ilearn:init` — ROADMAP.md must exist in cwd.
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn:init` first to create one." and stop.

## Flow

### Step 1: Detect workspace

Check if `ROADMAP.md` exists in cwd. If not → warn and stop.

### Step 2: Identify concept

If a concept path argument was provided (e.g., `concepts/java-oop/`):
1. Verify the folder exists at `concepts/<slug>/`.
2. Verify the concept slug appears in ROADMAP.md (grep for `[[concepts/<slug>/]]`). If not → print "Concept `<slug>` is not in your ROADMAP.md." and stop.

If no argument was provided:
1. Read ROADMAP.md and parse all concept entries.
2. Present as a numbered list.
3. Ask user to pick one by number, or type the slug directly.

### Step 3: Check theory.md

1. Read `concepts/<slug>/theory.md`.
2. If theory.md is empty or only has placeholder text (`# <Name>\n\nStudy notes go here.`):
   - Print: "theory.md is empty. Please run `/ilearn:create-theory concepts/<slug>/` first to populate theory notes." and stop.

### Step 4: Check existing questions

1. Read `concepts/<slug>/questions.md`.
2. If questions.md has content (not empty, not just HTML comments):
   - Count the questions.
   - Print: "questions.md already has N questions. Overwrite? [y/N]"
   - If user says no → stop.

### Step 5: Generate questions

1. Read `.claude/plugins/ilearn/references/workspace-format.md` to understand the workspace structure and questions.md format.
2. Read `.ilearn/config.json` to get `current_level` and `target_level`.
3. Spawn a general-purpose agent (`subagent_type: "general-purpose"`, `run_in_background: false`). Pass as prompt:

```
You are a question writer for technical interview preparation.

TOPIC: <Concept Name>
THEORY:
<content of theory.md>

Read .claude/plugins/ilearn/references/workspace-format.md first to understand the workspace structure and questions.md format.

Generate 5-10 interview-style review questions based solely on the theory content provided. Cover key concepts, edge cases, trade-offs, and practical applications appropriate for a <current_level> → <target_level> interview.

Return as a numbered markdown list:
1. What is ...?
2. Explain ...
3. How would you ...?
```

4. The agent returns a numbered list of questions.
5. If the generated list is empty or the agent fails → print "Failed to generate questions. Try again or provide questions manually." and stop.

### Step 6: Write questions.md

Write the numbered questions to `concepts/<slug>/questions.md`.

### Step 7: Summary

Print:

```
✅ Questions generated for <Concept Name>

File: concepts/<slug>/questions.md
Questions: <N>

Next steps:
  /ilearn:review concepts/<slug>/   — Answer these questions and get scored
```

### Step 8: Commit

```bash
git add concepts/<slug>/questions.md
git commit -m "feat: questions — <Concept Name>"
```

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn:init` first.
- **Concept not in ROADMAP.md**: Refuse.
- **theory.md empty**: Prompt user to run create-theory first — refuse to generate blind.
- **questions.md exists**: Confirm overwrite before proceeding.
- **Agent returns empty list**: Show failure, ask user to retry or add questions manually.
- **Config file missing**: Use defaults "Junior" → "Mid" if .ilearn/config.json is not readable.
