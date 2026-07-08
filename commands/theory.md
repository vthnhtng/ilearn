---
name: theory
description: "Deep-dive interactive lesson on a concept. Reads existing theory.md, delivers an interactive teaching session via teacher sub-agent, appends new insights to theory.md."
---

# ilearn-theory — Interactive Concept Lesson

## When to use
User types `/ilearn:theory` or `/ilearn:theory concepts/<slug>/` to get an interactive lesson on a concept.

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
2. Present as a numbered list — show name and current status (reviewed score or new).
3. Ask user to pick one by number, or type the slug directly.

### Step 3: Read theory

1. Read `concepts/<slug>/theory.md`.
2. If theory.md is empty or only has placeholder text → note it as empty.

### Step 4: Ask what the user wants to learn

1. Ask: "What do you want to learn about `<Concept Name>`?"
   - Options: "Give me an introduction", or specify a specific question/topic.
2. If they pick "introduction" → set user_request = "introduction".
3. If they ask a specific question → set user_request to their exact question.

### Step 5: Run the lesson

1. Read `.claude/plugins/ilearn/references/workspace-format.md` to recall format rules.
2. Spawn a **teacher** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "teacher"`, `run_in_background: false`). Pass as prompt:

```
You are teacher. Here are the inputs:

TOPIC: <Concept Name>
CURRENT_LEVEL: <current_level from .ilearn/config.json>
TARGET_LEVEL: <target_level from .ilearn/config.json>
THEORY: <content of theory.md or "empty">

USER_REQUEST: <user's question or "introduction">

Read .claude/plugins/ilearn/references/workspace-format.md first, then
.claude/plugins/ilearn/agents/teacher.md for your instructions.

Begin the lesson now.
```

3. The teacher runs the interactive session — teach → check → follow-up → more depth — until the user signals done.
4. At the end, the teacher returns the `## Teacher's Notes` block.

### Step 6: Append insights to theory.md

1. Parse the returned Teacher's Notes block.
2. Append it to `concepts/<slug>/theory.md`:
   - If theory.md currently only has the placeholder (`# <Concept Name>\n\nStudy notes go here.`), replace with the Teacher's Notes content wrapped in the topic heading.
   - Otherwise, append the Teacher's Notes after `---` separator before the existing content:

```
<existing theory.md content>

---

<Teacher's Notes content>
```

### Step 7: Summary

Print summary:

```
✅ Lesson complete!

Concept:     <Concept Name>
New insights appended to theory.md

Next steps:
  /ilearn:review concepts/<slug>/   — Test your understanding
  /ilearn:theory concepts/<slug>/   — Dive deeper on a specific aspect
```

### Step 8: Commit

```bash
git add concepts/<slug>/theory.md
git commit -m "feat: theory lesson — <Concept Name>"
```

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn:init` first.
- **Concept not in ROADMAP.md**: Refuse — "Concept `<slug>` is not in your ROADMAP.md."
- **Empty theory.md**: Teacher teaches from scratch using topic name + web search.
- **Teacher doesn't return notes block**: Show raw output, ask user what insights to keep.
- **User walks away mid-lesson**: If teacher agent times out or returns nothing, print "Lesson interrupted. No changes saved."
- **Repeated lessons**: Teacher's Notes accumulate in theory.md — each session appends with a new `---` separator.
