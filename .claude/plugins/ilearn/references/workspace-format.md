# ilearn Workspace Format

This document defines the exact file structure and format rules for ilearn workspaces.
Every ilearn sub-agent MUST read this before reading or writing workspace files.

## Directory Tree

```
<workspace-root>/
├── README.md                  # Workspace overview (free-form, user-editable)
├── ROADMAP.md                 # Master checklist — see format below
├── concepts/                  # One subfolder per concept
│   └── <concept-slug>/        # kebab-case slug matching the concept name
│       ├── theory.md          # Learning notes (user or AI populated)
│       ├── questions.md       # Review questions (one per line numbered)
│       ├── answers.md         # User's answers ### Q1, ### Q2 aligned to questions
│       └── review.md          # YAML frontmatter + markdown feedback (see below)
├── interviews/                # Mock interview logs
└── .ilearn/
    └── config.json            # Plugin metadata
```

## ROADMAP.md Format

### Headers (lines 1-8 approximates)

```markdown
# Roadmap: <Topic>

**Current Level:** <level> → **Target Level:** <level>
**Concepts:** <completed>/<total> (<percentage>%)
**Overall Score:** <score>/10
**Last Updated:** <YYYY-MM-DD>
```

### Concept entries

```markdown
## <Category Name>

- [ ] <Concept Name> | [[concepts/<concept-slug>/]]
- [x] <Concept Name> | Score: <N/10> | [[concepts/<concept-slug>/]] | Reviewed: <YYYY-MM-DD>
```

Rules:
- `[x]` = concept reviewed and scored. `[ ]` = not yet reviewed.
- `| Score: N/10 |` only on `[x]` lines.
- `[[concepts/<slug>/]]` is an Obsidian-compatible wikilink.
- Categories are `##` headings. Order implies priority.

## Concept Subfolder Files

### theory.md
Free-form markdown. No required structure. May be empty.

### questions.md
```markdown
1. What is ...?
2. Explain ...
```

One question per numbered line. Groups with headers allowed.

### answers.md
```markdown
### Q1
<answer text>

### Q2
<answer text>
```

Answers indexed to questions by number. May be empty (user answers via chat).

### review.md
```yaml
---
status: new              # new | in_progress | reviewed
score: 0                 # 0-10 overall. 0 if unreviewed.
breakdown:
  understanding: 0       # 1-10
  depth: 0              # 1-10
  communication: 0      # 1-10
attempt: 0              # incremented each review
reviewed_at: ""          # YYYY-MM-DD, empty if unreviewed
---

## Feedback

<qualitative feedback, empty if unreviewed>
```

`status: new` = never reviewed. `in_progress` = attempted but score < 7. `reviewed` = passed.

## Interview Log Format

File: `interviews/<date>_<topic-slug>_<score>.md`

```markdown
# Interview Log

**Date:** <YYYY-MM-DD>
**Topic:** <topic>
**Duration:** <minutes> min
**Concepts Covered:** [[concepts/<slug>/]], ...
**Result:** PASS | Overall Score: <N/10>
...
```

## .ilearn/config.json

```json
{
  "topic": "Java Spring Boot Backend Developer",
  "current_level": "Junior",
  "target_level": "Middle",
  "ilearn_version": "1.0.0",
  "created_at": "2026-07-07T17:00:00Z",
  "roadmap_source": "auto"
}
```

`roadmap_source`: `"auto"` or `"manual"`.
