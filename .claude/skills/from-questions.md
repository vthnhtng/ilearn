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
