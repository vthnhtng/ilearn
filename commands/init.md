---
name: init
description: "Initialize a new ilearn workspace for interview preparation. Scaffolds folder structure, ROADMAP.md, and concept subfolders. Supports auto-generated roadmaps via deep research or manual user-provided roadmaps."
---

# ilearn-init — Initialize Interview Training Workspace

## When to use
User types `/ilearn:init` or `/ilearn:init "<topic>"` to create a new interview preparation workspace.

## Prerequisites
- Run this in the directory where you want the workspace created.
- If `ROADMAP.md` already exists in cwd, WARN the user and ask for confirmation before re-initializing.

## Flow

### Step 1: Gather Input

1. If no topic argument was provided, ask: "What role or topic do you want to prepare for? (e.g., 'Java Spring Boot Backend Developer')"
2. Ask: "What is your current level?" — offer choices: Fresh, Junior, Mid, Senior
3. Ask: "What is your target level?" — offer choices: Junior, Mid, Senior
4. Ask: "How should we create the roadmap?" — options:
   - **Auto**: I research and generate a structured roadmap for you
   - **Manual**: I create an empty ROADMAP.md and folder structure — you fill in the concepts

5. IF auto → ask: "What is your target timeline?" — choices: 1 month, 3 months, 6 months, Any

### Step 2: Generate Roadmap (Auto Mode)

If user chose auto:

1. Read `.claude/plugins/ilearn/references/workspace-format.md` to recall the format rules.
2. Spawn a **roadmap-researcher** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "roadmap-researcher"`, `run_in_background: false`). Pass as prompt:

```
You are roadmap-researcher. Here are the inputs:

TOPIC: <topic>
CURRENT_LEVEL: <current_level>
TARGET_LEVEL: <target_level>
TIMELINE: <timeline>

Read .claude/plugins/ilearn/references/workspace-format.md first, then
.claude/plugins/ilearn/agents/roadmap-researcher.md for your instructions.

Produce the concept list in the required YAML format.
```

3. Parse the YAML output. If parsing fails, show the raw output to the user and ask them to help format it.
4. Present the roadmap to the user as a formatted list:

```
I've researched and prepared this roadmap for <topic>:

## 1. Core Java Foundation
- Java OOP (4h)
- Collections Framework (3h)

## 2. Spring Framework
- Spring IOC / DI (5h)

Total: <N> concepts across <M> categories

Does this look good? You can ask me to:
- Add/remove concepts
- Reorder categories
- Regenerate
- Or approve to scaffold the workspace
```

5. Handle user edit requests. Accept instructions like "remove X", "add Y", "move Z to top". Keep iterating until user approves.

### Step 3: Manual Roadmap Mode

If user chose manual:

1. Ask the user to provide the ROADMAP.md content directly (paste into chat or write to file).
2. If the user pastes content → parse and validate it. If the user says "I'll write it later" → create a placeholder ROADMAP.md with only headers.
3. Proceed to scaffolding.

### Step 4: Scaffold Workspace

After the roadmap content is finalized (from auto mode or manual), scaffold the workspace:

#### 4a. Create `README.md`

```markdown
# <topic>

**Status:** Active
**Current Level:** <current_level>
**Target Level:** <target_level>
**Created:** <current_date>

---

This workspace is managed by ilearn. Use `/ilearn:review` to review concepts
and `/ilearn:interview` for mock interviews.
```

#### 4b. Create `ROADMAP.md`

If auto mode, write the approved concepts with this format. The header block must match exactly.

For the ROADMAP.md wikilinks, use numbered two-level nesting: `[[concepts/<NN-category-slug>/<NN-concept-slug>/]]`

- `NN-category-slug` is the category's slug prefixed with a two-digit zero-padded sequential number (01, 02...) in the order categories appear in the YAML.
- `NN-concept-slug` is the concept's slug prefixed with a two-digit zero-padded sequential number (01, 02...) per-category in the order concepts appear under that category.

```markdown
# Roadmap: <topic>

**Current Level:** <current_level> → **Target Level:** <target_level>
**Concepts:** 0/<total> (0%)
**Overall Score:** —/10
**Last Updated:** <current_date>

---

## <Category Name>

- [ ] <Concept Name> | [[concepts/<NN-category-slug>/<NN-concept-slug>/]]

## <Next Category>

- [ ] <Concept Name> | [[concepts/<NN-category-slug>/<NN-concept-slug>/]]
```

The section after `---` is one `## <Category Name>` per category. Each concept under it is `- [ ] <Concept Name> | [[concepts/<NN-category-slug>/<NN-concept-slug>/]]`.

If manual mode and the user provided content, write their content. If the pasted content uses the old flat `[[concepts/<slug>/]]` format, ask the user if they want to convert to the new nested format. If yes, derive category slugs from `## <Category Name>` headings, assign NN-prefixes, and rewrite entries. If manual and no content yet, write the headers-only template:

```markdown
# Roadmap: <topic>

**Current Level:** <current_level> → **Target Level:** <target_level>
**Concepts:** 0/0 (0%)
**Overall Score:** —/10
**Last Updated:** <current_date>

---

<!-- Add your roadmap categories and concepts below -->

## Category 1

- [ ] Concept 1 | [[concepts/01-category-1/01-concept-1/]]
```

#### 4c. Create concept subfolders

For each concept parsed from the roadmap, create the two-level folder structure:

1. First, create `concepts/<NN-category-slug>/` for each category (if it doesn't exist yet).
2. Then, for each concept, create `concepts/<NN-category-slug>/<NN-concept-slug>/`.

Use the same NN prefixes assigned in ROADMAP.md (Step 4b). If a category has zero concepts, create the category directory but skip concept subdirectories. Each concept subfolder gets exactly four files:

**`concepts/<NN-category-slug>/<NN-concept-slug>/theory.md`:**
```markdown
# <Concept Name>

Study notes go here.
```

**`concepts/<NN-category-slug>/<NN-concept-slug>/questions.md`:**
```markdown
<!-- Questions will be added here during review sessions -->
```

**`concepts/<NN-category-slug>/<NN-concept-slug>/answers.md`:**
```markdown
<!-- Your answers go here -->
```

**`concepts/<NN-category-slug>/<NN-concept-slug>/review.md`:**
```markdown
---
status: new
score: 0
breakdown:
  understanding: 0
  depth: 0
  communication: 0
attempt: 0
reviewed_at: ""
---
```

#### 4d. Create `interviews/` directory with `.gitkeep`

```
interviews/.gitkeep
```

#### 4e. Create `.ilearn/config.json`

```json
{
  "topic": "<topic>",
  "current_level": "<current_level>",
  "target_level": "<target_level>",
  "ilearn_version": "1.0.0",
  "created_at": "<ISO timestamp>",
  "roadmap_source": "auto|manual"
}
```

Set `"roadmap_source"` to `"auto"` or `"manual"` based on user's choice.

### Step 5: Git Init & Commit

1. `git init` (if not already a git repo — check with `git rev-parse --git-dir` first)
2. `git add .`
3. `git commit -m "feat: initialize ilearn workspace for <topic>"`

### Step 6: Summary

Report to the user:

```
✅ ilearn workspace initialized successfully!

Topic:       <topic>
Level:       <current_level> → <target_level>
Concepts:    <N>
Roadmap:     <auto/manual>

Next steps:
  /ilearn:review         — Start reviewing concepts
  /ilearn:interview      — Mock interview (after reviewing concepts)
```

## Edge Cases

- **Already a workspace**: If `ROADMAP.md` exists in cwd, print a warning and ask for confirmation before proceeding.
- **Empty topic**: Ask until non-empty.
- **roadmap-researcher returns bad YAML**: Show raw output, ask user to help format it, or offer to regenerate.
- **User wants to redo**: If user says "no" during approval, ask what to change. Don't force-proceed.
