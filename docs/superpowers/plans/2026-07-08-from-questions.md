# From-Questions Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `/ilearn:from-questions` command and a "From Questions File" option in `/ilearn:init` that reads a `.txt`/`.md` file of interview questions, classifies them into categories/concepts, and scaffolds a workspace with pre-populated `questions.md` files.

**Architecture:** One new agent (questions-classifier) for grouping questions into YAML. One new skill (from-questions) for the orchestration flow. Init.md gains a 3rd roadmap option that delegates to the same flow.

**Tech Stack:** Plain Markdown + YAML — zero dependencies.

## Global Constraints

- All files are Markdown with YAML frontmatter — no JS/Python
- Agent output format must match roadmap-researcher YAML shape with additional `questions: [string]` per concept
- Scaffold logic (4a-4e) is already defined in `init.md` — reuse by reference, not duplication
- Command file goes in `commands/` with matching `.claude/skills/` file
- theory.md stays as placeholder (no AI-generated theory) — that's what `/ilearn:theory` is for

---

### Task 1: Create questions-classifier agent

**Files:**
- Create: `.claude/plugins/ilearn/agents/questions-classifier.md`

**Interfaces:**
- Consumes: raw questions text from a file
- Produces: YAML with `categories[].concepts[].questions: [string]`

- [ ] **Step 1: Write the agent file**

```markdown
---
name: questions-classifier
description: "Reads a raw list of interview questions and groups them into categorized concepts, outputting structured YAML with questions assigned per concept."
---

# questions-classifier

You are a senior technical curriculum designer. Your job is to group interview questions into a structured concept roadmap.

## Input

You receive a raw text block of interview questions, one per line or as markdown list items.

## Your Task

1. Read through all the questions
2. Group them into categories (max 8 categories) by topic
3. Within each category, group related questions into individual concepts
4. Order: foundational first, interview-high-frequency first
5. Each concept name should be a concrete single topic that the questions cover

## Output Format

Return ONLY a valid YAML block wrapped in ```yaml:

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

### Rules
- Category `slug`: kebab-case, lowercase, match `^[a-z][a-z0-9-]+$`
- Concept `slug`: kebab-case, lowercase, match `^[a-z][a-z0-9-]+$`
- `difficulty`: one of `beginner`, `intermediate`, `advanced`
- `estimated_hours`: integer, rough study time
- `questions`: array of strings — the actual questions assigned to this concept
- Every question from the input must be assigned to exactly one concept
- If a question doesn't fit any category, put it in a "General" category
- Include 3-15 concepts depending on the number of questions
```

- [ ] **Step 2: Commit**

```bash
git add .claude/plugins/ilearn/agents/questions-classifier.md
git commit -m "feat: add questions-classifier agent for grouping interview questions"
```

---

### Task 2: Create from-questions command and skill

**Files:**
- Create: `commands/from-questions.md`
- Create: `.claude/skills/from-questions.md`

**Interfaces:**
- Consumes: questions-classifier agent, workspace-format.md reference, scaffold pattern from init.md
- Produces: full ilearn workspace from a questions file

- [ ] **Step 1: Create command file `commands/from-questions.md`**

```markdown
---
name: from-questions
description: "Create a roadmap and workspace from an interview questions file (.txt/.md). Automatically groups questions into categories and concepts."
---

# ilearn:from-questions — Create Workspace from Questions File

## When to use
User types `/ilearn:from-questions` or `/ilearn:from-questions <file-path>` to generate a workspace from a questions file.
```

- [ ] **Step 2: Create skill file `.claude/skills/from-questions.md`**

```markdown
---
name: from-questions
description: "Scaffold an ilearn workspace from an interview questions file. Reads .txt/.md, classifies questions into categories/concepts, scaffolds folders with pre-populated questions."
---

# ilearn:from-questions — Generate Workspace from Questions File

## When to use
User types `/ilearn:from-questions` or `/ilearn:from-questions <path>` to create a workspace from a questions file.

## Prerequisites
- Run this in the directory where you want the workspace created.
- If `ROADMAP.md` already exists in cwd, WARN the user and ask for confirmation before re-initializing.

## Flow

### Step 1: Get the questions file path

1. If a path argument was provided, use it. Otherwise ask: "What is the path to your questions file? (.txt or .md)"
2. Validate the file exists. If not found, ask again or offer to let the user paste the questions directly.
3. If the path is a directory, scan it for `.txt` and `.md` files and let the user pick one.
4. Read the file content.

### Step 2: Validate content

1. If the file is empty, show an error: "The file is empty. Please provide a file with interview questions."
2. If the file has fewer than 3 lines that look like questions, warn: "This file has very few questions (<3). The generated roadmap may not be meaningful."
3. If the file has more than 100 questions, warn: "This file has a large number of questions (>100). Consider using a more focused set for better results."

### Step 3: Get topic and levels

1. Ask: "What role or topic do these questions cover? (e.g., 'Java Spring Boot Backend Developer')"
2. Ask: "What is your current level?" — Fresh, Junior, Mid, Senior
3. Ask: "What is your target level?" — Junior, Mid, Senior

### Step 4: Classify questions

1. Read `.claude/plugins/ilearn/references/workspace-format.md` to recall format rules.
2. Spawn the **questions-classifier** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "questions-classifier"`, `run_in_background: false`). Pass as prompt:

```
You are questions-classifier. Here are the raw interview questions:

<content of the questions file>

Read .claude/plugins/ilearn/agents/questions-classifier.md for your instructions.

Produce the categorized YAML output.
```

3. Parse the YAML output. If parsing fails, show raw output and ask user to help format it.

### Step 5: Present to user

Show the classified roadmap:

```
I've grouped your questions into this roadmap for <topic>:

## 1. Java Core
- OOP Concepts (3h) — 3 questions
- Collections (4h) — 2 questions

## 2. Spring Framework
- Dependency Injection (3h) — 4 questions

Total: <N> concepts across <M> categories, <Q> questions

Does this look good? You can ask me to:
- Rename or merge categories
- Move questions between concepts
- Add/remove concepts
- Regenerate
- Or approve to scaffold the workspace
```

### Step 6: Handle edit requests

Accept instructions like "merge X into Y", "rename Z", "move question 5 to concept A". Keep iterating until user approves.

### Step 7: Scaffold workspace

After user approves, scaffold using the same pattern as init.md Step 4 (4a-4e), with these differences:

- `roadmap_source` in config.json is set to `"from-questions"`
- Each concept's `questions.md` is pre-populated with its assigned questions (numbered):

```markdown
1. Explain polymorphism with examples
2. What is the difference between abstraction and encapsulation?
```

- theory.md stays as placeholder (`# <Concept Name>\n\nStudy notes go here.`)
- answers.md stays as placeholder (`<!-- Your answers go here -->`)

### Step 8: Git Init & Commit

Same as init.md Step 5.

### Step 9: Summary

```
✅ Workspace created from questions file!

Topic:       <topic>
Level:       <current_level> → <target_level>
Concepts:    <N>
Questions:   <Q>
Roadmap:     from-questions

Next steps:
  /ilearn:review         — Start reviewing concepts
  /ilearn:interview      — Mock interview (after reviewing concepts)
```

## Edge Cases

- **File not found**: Ask for valid path or offer paste fallback.
- **Empty file / no parseable questions**: Error with explanation.
- **Too many questions (>100)**: Warn but proceed.
- **Too few questions (<3)**: Warn but proceed.
- **Already a workspace (ROADMAP.md exists)**: Warn + confirm before proceeding.
- **Path is a directory**: Scan for .txt/.md files inside, let user pick.
- **Malformed YAML from agent**: Show raw output, ask user to help format.
- **User cancels mid-way**: Clean up any partial files created.
```

- [ ] **Step 3: Commit**

```bash
git add commands/from-questions.md .claude/skills/from-questions.md
git commit -m "feat: add from-questions command and skill for generating workspace from questions file"
```

---

### Task 3: Update init.md with "From Questions File" option

**Files:**
- Modify: `.claude/skills/init.md`

- [ ] **Step 1: Add 3rd roadmap option in init.md Step 1**

In Step 1 item 4, add "From Questions File" as a third option:

Edit line 22-24 of `.claude/skills/init.md`:

Old:
```
4. Ask: "How should we create the roadmap?" — options:
   - **Auto**: I research and generate a structured roadmap for you
   - **Manual**: I create an empty ROADMAP.md and folder structure — you fill in the concepts

5. IF auto → ask: "What is your target timeline?" — choices: 1 month, 3 months, 6 months, Any
```

New:
```
4. Ask: "How should we create the roadmap?" — options:
   - **Auto**: I research and generate a structured roadmap for you
   - **From Questions File**: I read your interview questions file and build a roadmap from it
   - **Manual**: I create an empty ROADMAP.md and folder structure — you fill in the concepts

5. IF auto → ask: "What is your target timeline?" — choices: 1 month, 3 months, 6 months, Any
6. IF from questions file → delegate to the from-questions skill flow (Steps 1-6 of from-questions.md), then resume at Step 4 (scaffolding) with the classified YAML
```

- [ ] **Step 2: Add "from-questions" to Step 1 description frontmatter**

Edit the description line in `.claude/skills/init.md` frontmatter:
Old: `Supports auto-generated roadmaps via deep research or manual user-provided roadmaps.`
New: `Supports auto-generated roadmaps via deep research, from-questions files, or manual user-provided roadmaps.`

- [ ] **Step 3: Add Step 1 item 6 as described above**

Insert after the existing Step 1:
```
6. IF from questions file → follow the from-questions flow:
   1. Get the questions file path (ask user)
   2. Validate and read the file
   3. Spawn questions-classifier agent
   4. Present roadmap to user for approval
   5. Handle edit requests
   6. Resume at Step 4 (scaffolding) with the approved YAML
```

- [ ] **Step 4: Update config.json roadmap_source values in 4e**

In line 205, change `"roadmap_source": "auto|manual"` to `"roadmap_source": "auto|manual|from-questions"`

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/init.md
git commit -m "feat: add from-questions-file option to ilearn:init"
```

---

### Task 4: Update plugin.json

**Files:**
- Modify: `plugin.json`

- [ ] **Step 1: Add new files to plugin.json**

Add to the `files` array:
```json
    ".claude/skills/from-questions.md",
    ".claude/plugins/ilearn/agents/questions-classifier.md"
```

- [ ] **Step 2: Commit**

```bash
git add plugin.json
git commit -m "chore: register from-questions skill and questions-classifier agent in plugin manifest"
```

---

### Task 5: Verify the feature works

**Files:**
- None — run-time verification

- [ ] **Step 1: Install the plugin locally**

```bash
# From the ilearn repo root
claude plugin install .
```

- [ ] **Step 2: Create a test questions file**

```bash
mkdir -p /tmp/ilearn-test
cat > /tmp/ilearn-test/java-questions.md << 'EOF'
1. What is the difference between == and equals() in Java?
2. Explain how HashMap works internally
3. What is the difference between abstract class and interface?
4. How does garbage collection work in Java?
5. What is the difference between ArrayList and LinkedList?
6. Explain the Spring IoC container
7. What is dependency injection and what are the types?
8. Explain the Spring Boot auto-configuration
9. What is the difference between @Component, @Service, and @Repository?
10. How does Spring Security filter chain work?
EOF
```

- [ ] **Step 3: Test in a temporary directory**

```bash
mkdir -p /tmp/ilearn-test-workspace && cd /tmp/ilearn-test-workspace
# Run the from-questions command
# Expected: should ask for file path, classify questions, present roadmap, scaffold workspace
```

- [ ] **Step 4: Clean up test files**

```bash
rm -rf /tmp/ilearn-test /tmp/ilearn-test-workspace
```
