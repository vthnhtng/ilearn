---
name: review
description: "Score answers for a concept. Reads answers.md, theory.md, questions.md and runs the reviewer agent. If answers.md is empty, tells user to write answers first."
---

# ilearn-review — Score Your Answers

## When to use
User types `/ilearn:review` or `/ilearn:review concepts/<slug>/` to get their answers scored. Write your answers in `answers.md` first, then run this command.

## Prerequisites
- A workspace initialized with `/ilearn:init` — ROADMAP.md must exist in cwd.
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn:init` first to create one." and stop.

## Flow

### Step 1: Detect workspace

Check if `ROADMAP.md` exists in cwd. If not → warn and stop.

### Step 2: Identify concept

If a concept path argument was provided (e.g., `concepts/java-oop/`):
1. Verify the folder exists at `concepts/<slug>/` with the required files (`theory.md`, `questions.md`).
2. Verify the concept slug appears in ROADMAP.md (grep for `[[concepts/<slug>/]]`). If not → print "Concept `<slug>` is not in your ROADMAP.md. Add it to ROADMAP.md first." and stop.
3. Check if `concepts/<slug>/review.md` exists. If it does, read its frontmatter:
   - If `status: reviewed` and score >= 7, print: "Concept `<slug>` is already reviewed (score: N/10). Re-review? [y/N]"
   - Wait for user response. If response is not 'y' or 'yes', stop.

If no argument was provided:
1. Read ROADMAP.md and parse all concept entries.
2. Filter for concepts with status `new` or `in_progress`:
   - Check each concept's `concepts/<slug>/review.md` for status field.
   - If `review.md` missing or `status: new` → show as "New"
   - If `status: in_progress` → show as "In Progress (needs retry, score: <N>/10)"
3. Present as a numbered list.
4. Ask user to pick one by number, or type the slug directly.

### Step 3: Check answers.md

1. Read `concepts/<slug>/answers.md`.
2. If answers.md is empty or only has placeholder content (just HTML comments):
   - Print: "answers.md is empty. Write your answers to `concepts/<slug>/answers.md` first (use `### Q1`, `### Q2` etc. matching questions.md), then run `/ilearn:review` again."
   - Stop.

### Step 4: Check prerequisites

1. Read `theory.md` from the concept folder.
   - If theory.md is empty or placeholder → print: "theory.md is empty. Run `/ilearn:create-theory concepts/<slug>/` first to populate theory notes." and stop.
2. Read `questions.md` from the concept folder.
   - If questions.md is empty or just HTML comments → print: "questions.md is empty. Run `/ilearn:create-question concepts/<slug>/` first to generate review questions." and stop.
3. Present the sources to the user: "Reviewing answers for <Concept Name> using <N> questions from questions.md."

### Step 5: Score via reviewer sub-agent

1. Read the current `review.md` from the concept folder to get the current `attempt` count:
   - If `review.md` exists and has `attempt: N` → new attempt = N + 1
   - If `review.md` missing → attempt = 1
   - Also extract the old feedback body (content after the YAML frontmatter, i.e., after the closing `---` line) for archiving.
2. Spawn a **reviewer** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "reviewer"`, `run_in_background: false`). Pass as prompt:

```
You are reviewer. Here are the inputs:

TOPIC: <concept name>
THEORY:
<content of theory.md>

QUESTIONS:
<content of questions.md>

ANSWERS:
<content of answers.md>

attempt: <attempt_number>

Read .claude/plugins/ilearn/references/workspace-format.md first, then
.claude/plugins/ilearn/agents/reviewer.md for your instructions.

Produce a complete review.md file content.
```

3. Parse the returned review.md content:
   - Extract score, breakdown, status from YAML frontmatter.
   - Extract feedback body.
   - If parsing fails → print raw output and ask user to help interpret.
4. Set `reviewed_at` to current date (YYYY-MM-DD), replacing the date the reviewer returned.
5. Write the full content to `concepts/<slug>/review.md`. If old feedback was extracted in step 5.1, append it under `## Previous Feedback` before writing, so the final structure is: `<YAML frontmatter>\n\n## Feedback\n<new feedback>\n\n## Previous Feedback\n<old feedback>`

### Step 6: Update ROADMAP.md

1. Find the concept line in ROADMAP.md matching `[[concepts/<slug>/]]`.
2. Replace the line:

```
- [ ] <Concept Name> | [[concepts/<slug>/]]
```

with:

```
- [x] <Concept Name> | Score: <overall>/10 | [[concepts/<slug>/]] | Reviewed: <YYYY-MM-DD>
```

3. Update the header block:
   - Increment `<completed>` count if concept was previously `[ ]`.
   - Update percentage: `<completed>/<total> (<percentage>%)`.
   - If this is the first reviewed concept, set `Overall Score: <score>/10`.
   - If there are already reviewed concepts, recalculate average of all concept scores in ROADMAP.md.
   - Update `Last Updated:` to today's date.

### Step 7: Report & Commit

1. Print summary:

```
✅ Review complete!

Concept:     <Concept Name>
Score:       <overall>/10
Status:      <reviewed / in_progress (needs retry)>
Attempt:     <attempt_number>

Breakdown:
  Understanding:  <U>/10
  Depth:          <D>/10
  Communication:  <C>/10

<If status is in_progress: The score is below 7. Review the feedback and try again with /ilearn:review concepts/<slug>/>
<If status is reviewed: Great job! Move to the next concept or try /ilearn:interview for a mock interview.>
```

2. Git add and commit:

```bash
git add concepts/<slug>/
git add ROADMAP.md
git commit -m "feat: review <Concept Name> — score <overall>/10 (attempt <N>)"
```

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn:init` first.
- **Concept not in ROADMAP.md**: Refuse — "Concept `<slug>` is not in your ROADMAP.md. Add it first."
- **Multiple review attempts**: Increment `attempt` counter. Archive old feedback by keeping previous content below a `## Previous Feedback` section in review.md.
- **Empty answers.md**: Print message asking user to write answers to answers.md first — stop.
- **Empty theory.md**: Prompt user to run `/ilearn:create-theory` first — stop.
- **Empty questions.md**: Prompt user to run `/ilearn:create-question` first — stop.
- **review.md parsing fails**: Show raw reviewer output to user. Ask them to help extract scores.
- **ROADMAP.md has no `[ ]` or `[x]` concepts**: Print "No concepts found in ROADMAP.md." and stop.
