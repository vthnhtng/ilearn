# ilearn — AI-powered Technical Interview Training

ilearn is a plugin for [Claude Code](https://claude.ai) (and compatible AI coding agents) that creates structured workspaces for technical interview preparation. Define a target role/topic, and ilearn manages the full learning loop: roadmap generation → concept learning → Q&A review → mock interviews → progress tracking.

All data lives in plain Markdown files — zero external storage, fully portable, [Obsidian](https://obsidian.md)-compatible.

## Quick Start

ilearn ships as a **skills-directory plugin**: skills in `.claude/skills/` auto-load when Claude Code starts in this directory. No install step needed when running from here.

### Option A: Use in-place (cloned repo)

```bash
# Skills are already available. Just start a workspace:
/ilearn:init "Java Spring Boot Backend Developer"
```

### Option B: Install from local marketplace (use in other projects)

```bash
/plugin marketplace add ./
/plugin install ilearn@ilearn-dev

# Then in any project:
/ilearn:init "Your Topic"
```

### Option C: Install from GitHub

```bash
/plugin marketplace add vthnhtng/ilearn
/plugin install ilearn@ilearn-dev
```

## Skills

| Command | Description |
|---------|-------------|
| `/ilearn:init` | Scaffold a new workspace (auto or manual roadmap) |
| `/ilearn:review` | Review & score a concept (Understanding/Depth/Communication) |
| `/ilearn:interview` | Live mock interview session |
| `/ilearn:theory` | Interactive concept teaching |
| `/ilearn:status` | Progress overview with weak-area analysis |
| `/ilearn:obsidian-export` | Turn workspace into an Obsidian vault (config, dashboard, templates) |

### Bare aliases (when session starts in ilearn repo)

`/ilearn:init`, `/ilearn:review`, `/ilearn:interview`, `/ilearn:theory`, `/ilearn:status`, `/ilearn:obsidian-export` also work when `.claude/skills/` auto-loads.

## Workspace Structure

```
workspace/
├── README.md                  # Workspace overview
├── ROADMAP.md                 # Master checklist + progress
├── concepts/                  # One subfolder per concept
│   ├── java-oop/
│   │   ├── theory.md          # Learning notes
│   │   ├── questions.md       # Review questions
│   │   ├── answers.md         # Your answers
│   │   └── review.md          # AI review + score
│   └── ...
├── interviews/                # Mock interview logs
├── .ilearn/
│   └── config.json            # Workspace config
```

## Scoring Rubric

| Dimension | Weight | Scale |
|-----------|--------|-------|
| Understanding | 40% | 1-10 |
| Depth | 40% | 1-10 |
| Communication | 20% | 1-10 |
| **Overall** | **100%** | U×0.4 + D×0.4 + C×0.2 |

Score ≥ 7 = pass. Score < 7 = concept marked for retry.

## Files

| File | Purpose |
|------|---------|
| `plugin.json` | Legacy v1 manifest (`files` array for manual copying) |
| `.claude-plugin/plugin.json` | Plugin metadata (name, author, version) |
| `.claude-plugin/marketplace.json` | Marketplace catalog for `/plugin marketplace add` |
| `.claude/skills/*.md` | Skills — auto-loaded at session start |
| `commands/` | Same skills for plugin discovery via `source: "./"` |
| `agents/` | Sub-agent definitions for plugin discovery |

## Design

Full design spec: [`docs/ilearn/2026-07-07-ilearn-design.md`](docs/ilearn/2026-07-07-ilearn-design.md)

### Key Principles

- **Markdown-only** — all data in plain `.md` files, zero external storage
- **Obsidian-compatible** — wiki links, frontmatter, standard MD
- **Portable** — copy the workspace folder, everything moves
- **No dependencies** — no npm, no pip, no build step

## License

MIT
