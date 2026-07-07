# ilearn — AI Agent Guide

ilearn is a Claude Code plugin for structured technical interview preparation. It scaffolds a workspace (ROADMAP + concept folders), runs AI-driven Q&A reviews, conducts mock interviews, and tracks progress — all in plain Markdown.

## Architecture

- **Skills** (`.claude/skills/*.md`) define the command flows (`/ilearn-init`, `/ilearn-review`, etc.) — auto-loaded at session start
- **Sub-agents** (`.claude/plugins/ilearn/agents/*.md`) power multi-turn interactions (interviewer, reviewer, teacher)
- **Format contract** (`.claude/plugins/ilearn/references/workspace-format.md`) — **ALL agents must read before touching workspace files**
- **Plugin metadata** (`.claude-plugin/plugin.json`) — name, author, version for marketplace
- **Marketplace catalog** (`.claude-plugin/marketplace.json`) — enables `/plugin marketplace add ./`

## Workspace Contract

- ROADMAP.md headers and concept entry syntax are strict — see `workspace-format.md` for the exact format
- `review.md` uses YAML frontmatter with `status`, `score`, `breakdown`, `attempt`, `reviewed_at`
- Concepts live under `concepts/<slug>/` with 4 files: `theory.md`, `questions.md`, `answers.md`, `review.md`
- Interview logs at `interviews/<date>_<topic>_<score>.md`
- Plain Markdown + `[[wikilinks]]` throughout — natively Obsidian-compatible

## Contribution

- Skills must be self-contained `.md` files with YAML frontmatter (`name`, `description`)
- Add new skills to `plugin.json`'s `files` array
- Keep zero external dependencies — no npm, no pip
- Test via `plugin-install` before submitting
