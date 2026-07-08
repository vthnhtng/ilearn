---
name: interview
description: "Run a mock interview session. Generates questions from reviewed concepts, scores answers, provides debrief with PASS/FAIL verdict and writes interview log."
---

# ilearn-interview — Mock Interview Session

## When to use
User types `/ilearn:interview` to start a mock interview session.

## Prerequisites
- A workspace initialized with `/ilearn:init` — ROADMAP.md must exist in cwd.
- At least 1 concept must be reviewed (score >= 7).
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn:init` first to create one." and stop.

## Flow

### Step 1: Detect workspace

1. Check if `ROADMAP.md` exists in cwd. If not → warn and stop.
2. Check that `.ilearn/config.json` exists. If missing, print: "Workspace config not found at `.ilearn/config.json`. The workspace may be corrupted. Run `/ilearn:init` to reinitialize." and stop.

### Step 2: Check reviewed concepts

1. Read ROADMAP.md and parse all concept entries.
2. If ROADMAP.md has no `[ ]` or `[x]` entries at all, print "No concepts found in ROADMAP.md." and stop.
3. Filter for reviewed concepts: look for lines with `[x]` and parse `Score: <N>/10` from each line.
4. Only include concepts where Score >= 7. Skip concepts with Score < 7 (in_progress).
5. If zero reviewed (score >= 7) concepts found, print: "No reviewed concepts found. Review at least 1 concept first with `/ilearn:review`." and stop.
6. Show the user which concepts will be covered: "Interview will cover <N> reviewed concepts: <list of names>"

### Step 3: Determine interview parameters

1. Ask: "How long should the interview be?" — options: 15 min, 30 min, 45 min, 60 min, or Auto (calculated as `max(15, min(60, num_concepts * 5))` minutes, where num_concepts is the number of reviewed concepts with score >= 7)
2. Ask: "Would you like to include your background/CV? I can tailor questions to your experience." If yes:
   - Look for common CV filenames in cwd: `CV.md`, `cv.md`, `resume.md`, `Resume.md`, or ask user for the path.
   - If found, read the CV content.
   - If not found, ask user to paste their background info or skip.

### Step 4: Collect reviewed concept content

1. For each reviewed concept, read `concepts/<slug>/theory.md`.
2. Compile a CONCEPTS block with slug, name, and theory content.

### Step 5: Run the interview

1. Spawn an **interviewer** sub-agent via the Agent tool (`subagent_type: "general-purpose"`, `description: "interviewer"`, `run_in_background: false`). Pass as prompt:

```
You are interviewer. Here are the inputs:

TOPIC: <topic from .ilearn/config.json>
CURRENT_LEVEL: <current_level from .ilearn/config.json>
TARGET_LEVEL: <target_level from .ilearn/config.json>
DURATION: <selected_duration>
CONCEPTS:
  - <slug>: <name>
    theory: <content of theory.md>
  - <slug>: <name>
    theory: <content of theory.md>
CV: <CV content or "Not provided">

Read .claude/plugins/ilearn/references/workspace-format.md first, then
.claude/plugins/ilearn/agents/interviewer.md for your instructions.

Conduct the interview now. Ask the first question.
```

2. The interviewer will run the full interactive session: questions → answers → follow-ups → scoring.
3. At the end, the interviewer returns the complete interview log content.

### Step 6: Write interview report

1. Parse the returned log content.
2. Extract the overall score and result (PASS/FAIL).
3. Create the report folder: `interviews/<dd-mm-yyyy>/` (e.g., `interviews/07-07-2026/`). If it already exists, append a counter (`-v2`, `-v3`).
4. Write three files into the folder:

   **`overview.md`** — header metadata + summary section:
   ```
   # Interview Log

   **Date:** <YYYY-MM-DD>
   **Topic:** <topic>
   **Current Level:** <current_level>
   **Target Level:** <target_level>
   **Duration:** <minutes> min
   **Concepts Covered:** [[concepts/<slug>/]], ...
   **Result:** PASS | Overall Score: <N>/10

   ---

   ## Summary

   **Strengths:**
   - <strength 1>
   - <strength 2>

   **Weaknesses:**
   - <weakness 1>
   - <weakness 2>

   **Final Verdict:** PASS | Overall Score: <N>/10
   <actionable advice>
   ```

   **`questions.md`** — numbered questions only (no answers):
   ```
   1. <question 1 text>
   2. <question 2 text>
   ...
   ```

   **`answers.md`** — full Q&A with scores and evaluation:
   ```
   ### Q1
   **Question:** <question text>
   **Difficulty:** <easy/medium/hard>
   **Concepts:** [[concepts/<slug>/]]
   **Answer:** <candidate's answer>
   **Score:** <N>/10
   **Evaluation:** <what was good, what was missing>
   **Follow-up:** <any follow-up asked, or N/A>

   ### Q2
   ...
   ```

5. Print summary:

```
✅ Interview complete!

Topic:       <topic>
Duration:    <minutes> min
Questions:   <N>
Score:       <N>/10
Result:      PASS / FAIL

Strengths:
- <strength 1>
- <strength 2>

Weaknesses:
- <weakness 1>
- <weakness 2>
```

### Step 7: Optionally Update Reviewed Concepts

After the interview, for each concept covered:
1. Compare the interview's per-question score for that concept against the concept's `review.md` score.
2. If the interview score differs by more than 2 points from the review score, append a note to `review.md`:
   ```
   ## Interview Note
   Score during interview on <YYYY-MM-DD>: <N>/10
   ```
3. This is optional — the main review scores come from `/ilearn:review`.

### Step 8: Commit

```bash
git add interviews/<dd-mm-yyyy>/
git commit -m "feat: interview — <topic> (<overall>/10, <result>)"
```

## Edge Cases

- **No ROADMAP.md**: Warn and stop — run `/ilearn:init` first.
- **No .ilearn/config.json**: Print "Workspace config not found at `.ilearn/config.json`. The workspace may be corrupted. Run `/ilearn:init` to reinitialize." and stop.
- **Zero reviewed concepts**: If concepts exist but none have score >= 7, print "No reviewed concepts found. Review at least 1 concept first with `/ilearn:review`." and stop.
- **No concept entries**: If ROADMAP.md has no `[ ]` or `[x]` entries, print "No concepts found in ROADMAP.md." and stop.
- **Missing or empty theory.md**: If theory.md is missing or empty, pass empty theory — interviewer falls back to topic name.
- **Interviewer doesn't finish**: If the sub-agent returns incomplete content or no log, show raw output and ask user to help.
- **CV file not found**: If user says yes to CV but no file found, ask them to paste or skip.
- **Duplicate folder names**: If `interviews/<dd-mm-yyyy>/` already exists, append `-v2`, `-v3` etc.
