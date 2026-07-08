---
name: create-theory
description: "Research and auto-write theory.md for a concept (or all concepts) using web search. Populates theory.md with comprehensive notes from a researcher agent."
---

# ilearn-create-theory — Auto-Write Theory Notes

## When to use
User types `/ilearn:create-theory [concepts/<slug>/ | --all]` to research and write theory notes for one or all concepts.

## Prerequisites
- A workspace initialized with `/ilearn:init` — ROADMAP.md must exist in cwd.
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn:init` first to create one." and stop.

## Flow

### Step 1: Detect workspace

Check if `ROADMAP.md` exists in cwd. If not → warn and stop.

### Step 2: Identify concept(s)

If argument is `--all`:
1. Scan ROADMAP.md for all concept entries.
2. For each concept, check if `concepts/<slug>/theory.md` exists and has substantive content (not just placeholder `# Title\n\nStudy notes go here.`).
3. Build a list of concepts with empty/placeholder theory.md.
4. If the list is empty → print "All concepts already have theory content." and stop.
5. Present the list with counts: "Found <N> concepts with empty theory.md. Process all? [y/N]"
6. If user says yes → process each concept one by one (loop through Step 3-6). Commit after each individual concept.
7. If user says no → stop.

If a concept path is provided (e.g., `concepts/java-oop/`):
1. Verify the folder exists at `concepts/<slug>/`.
2. Verify the concept slug appears in ROADMAP.md (grep for `[[concepts/<slug>/]]`). If not → print "Concept `<slug>` is not in your ROADMAP.md." and stop.
3. Proceed to Step 3.

If no argument was provided:
1. Read ROADMAP.md and parse all concept entries.
2. Present as a numbered list — show name and status.
3. Ask user to pick one by number, or type the slug directly.

### Step 3: Check existing content

1. Read `concepts/<slug>/theory.md` to check for existing content.
2. If theory.md has substantive content (not placeholder `# <Name>\n\nStudy notes go here.`):
   - Print: "theory.md already has content. Overwrite? [y/N]"
   - Save existing content as `old_content` for later preservation.
   - If user says no → skip this concept. If --all mode, move to next concept.
3. If empty/placeholder → proceed.

### Step 4: Research & write

1. Read `.claude/plugins/ilearn/references/workspace-format.md` to recall format rules.
2. Read `.claude/plugins/ilearn/agents/researcher.md` for agent instructions.
3. Read `concept name` from ROADMAP.md by parsing the concept entry line.
4. Read `.ilearn/config.json` to get `current_level` and `target_level`. If the file is not readable, use defaults "Junior" → "Mid".
5. Spawn a **researcher** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "researcher"`, `run_in_background: false`). Pass as prompt:

```
You are researcher. Here are the inputs:

TOPIC: <Concept Name>
CURRENT_LEVEL: <current_level from .ilearn/config.json>
TARGET_LEVEL: <target_level from .ilearn/config.json>

Read .claude/plugins/ilearn/references/workspace-format.md first, then
.claude/plugins/ilearn/agents/researcher.md for your instructions.

Produce theory.md content for this concept.
```

6. The researcher returns theory.md content as a markdown block.
7. If the agent times out or returns empty → print "Research failed for <Concept Name>. Skipping." In --all mode, continue to next concept.

### Step 5: Write file

1. If old_content was saved (user approved overwrite of substantive content), write:

```
<new theory content>

---

## Previous Version

<old_content>
```

2. If old content was just placeholder or empty, write the new content directly.
3. Write to `concepts/<slug>/theory.md`.

### Step 6: Summary (single concept)

Print:

```
✅ Theory written for <Concept Name>

File: concepts/<slug>/theory.md

Next steps:
  /ilearn:create-question concepts/<slug>/   — Generate review questions
  /ilearn:theory concepts/<slug>/            — Interactive lesson
  /ilearn:review concepts/<slug>/            — Test understanding
```

### Step 7: Commit (single concept)

```bash
git add concepts/<slug>/theory.md
git commit -m "feat: theory — <Concept Name>"
```

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn:init` first.
- **Concept not in ROADMAP.md**: Refuse — "Concept `<slug>` is not in your ROADMAP.md."
- **Empty/placeholder theory.md**: Proceed with research as normal.
- **Substantive theory.md exists**: Warn and ask for overwrite confirmation.
- **Web search fails**: Researcher falls back to LLM knowledge per its instructions.
- **Researcher times out**: Skip concept (--all) or report failure (single).
- **--all with nothing to fill**: "All concepts already have theory content."
- **--all interrupted mid-batch**: Each concept is committed individually, so partial progress is saved.
- **Config file missing**: Use defaults "Junior" → "Mid" if `.ilearn/config.json` is not readable.
