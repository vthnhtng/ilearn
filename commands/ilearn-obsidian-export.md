---
name: ilearn-obsidian-export
description: "Export an ilearn workspace as an Obsidian vault — adds .obsidian config, templates, dataview dashboards, and graph-view metadata for visual progress tracking."
---

# ilearn-obsidian-export — Export to Obsidian Vault

## When to use
User types `/ilearn-obsidian-export` to turn their ilearn workspace into a full Obsidian vault with visual dashboards, templates, and plugin config.

## Prerequisites
- A workspace initialized with `/ilearn-init` — ROADMAP.md must exist in cwd.
- If ROADMAP.md does not exist, print: "No ilearn workspace found in this directory. Run `/ilearn-init` first to create one." and stop.

## What this does

The workspace is already Obsidian-compatible (plain Markdown + `[[wikilinks]]`). This skill adds:

| What | Why |
|------|-----|
| `.obsidian/` config | App snippet, appearance, hotkeys, core plugins |
| Templates | Concept template, interview template for consistent creation |
| Dataview dashboard (`obsidian-dashboard.md`) | Live progress %, concept table, weak areas (requires Dataview plugin) |
| Graph-view tag metadata | Frontmatter tags on every concept file for filtered graph views |
| `README.md` update | Obsidian-specific launch instructions |

## Flow

### Step 1: Detect workspace

1. Check if `ROADMAP.md` exists in cwd. If not → warn and stop.
2. Check if `.obsidian/` already exists. If yes, warn: "This workspace already has Obsidian config. Overwrite? (y/N)" — only proceed on explicit "y" or "yes". Archive old config as `.obsidian.backup/` first.
3. Read `.ilearn/config.json` to get topic, levels.

### Step 2: Create `.obsidian/` directory

Create these files:

**`.obsidian/app.json`:**
```json
{
  "alwaysUpdateLinks": true,
  "newFileLocation": "folder",
  "newFileFolderPath": "concepts",
  "attachmentFolderPath": "assets",
  "useMarkdownLinks": false,
  "showUnsupportedFiles": false,
  "showFrontmatter": true,
  "spellcheck": true,
  "strictLineBreaks": true,
  "readableLineLength": true,
  "communityPluginSortOrder": "download"
}
```

**`.obsidian/appearance.json`:**
```json
{
  "accentColor": "#7c3aed",
  "cssTheme": "Minimal",
  "baseColor": 2,
  "enabledCssSnippets": ["ilearn-dashboard"]
}
```

**`.obsidian/hotkeys.json`:** (leave empty — `{}` — use defaults).

**`.obsidian/core-plugins.json`:**
```json
[
  "file-explorer",
  "global-search",
  "switcher",
  "graph",
  "backlink",
  "outgoing-link",
  "tag-pane",
  "page-preview",
  "templates",
  "note-composer",
  "command-palette",
  "editor-status",
  "starred",
  "markdown-importer",
  "word-count",
  "file-recovery"
]
```

**`.obsidian/community-plugins.json`:**
```json
[
  "dataview",
  "obsidian-tracker",
  "obsidian-minimal-settings"
]
```

**`.obsidian/snippets/ilearn-dashboard.css`:**
```css
/* ilearn dashboard card styling */
.ilearn-card {
  border: 1px solid var(--background-modifier-border);
  border-radius: 8px;
  padding: 1rem;
  margin: 1rem 0;
  background: var(--background-primary-alt);
}
.ilearn-card h2 { margin-top: 0; }
.ilearn-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 1rem;
}
.ilearn-stat {
  text-align: center;
  padding: 1rem;
}
.ilearn-stat .value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--accent-color);
}
.ilearn-stat .label {
  font-size: 0.85rem;
  color: var(--text-muted);
}
.tag-ilearn-reviewed { color: var(--color-green); }
.tag-ilearn-in-progress { color: var(--color-orange); }
.tag-ilearn-new { color: var(--color-blue); }
```

### Step 3: Add frontmatter tags to all concept files

For every concept folder under `concepts/`:

1. Read `concepts/<slug>/review.md` — parse the YAML frontmatter to get `status`, `score`.
2. Prepend/update `tags` field in each file:

**`concepts/<slug>/theory.md`:** Add frontmatter:
```yaml
---
tags: [ilearn/concept/<slug>, ilearn/status/<status>]
---
```

**`concepts/<slug>/review.md`:** Add `tags` to existing frontmatter:
```yaml
tags: [ilearn/concept/<slug>, ilearn/review]
```
Also add `score` and `status` fields if not already present (they should be, per workspace format).

3. Use Edit to inject the tags. If a file already has frontmatter, merge the `tags` array. If no frontmatter, prepend it.

### Step 4: Create templates

**`.obsidian/templates/concept.md`:**
```markdown
---
tags: [ilearn/concept/<%= slug %>, ilearn/status/new]
---

# <%= name %>

Study notes go here.

## Key Points

-

## Code Examples

```

**`.obsidian/templates/interview.md`:**
```markdown
---
tags: [ilearn/interview]
---

# Interview Log

**Date:** <%= date %>
**Topic:**
**Duration:** min
**Result:** | Overall Score: /10
```

### Step 5: Create Dataview Dashboard

Create **`obsidian-dashboard.md`** at workspace root:

```markdown
---
tags: [ilearn/dashboard]
---

# 📊 ilearn Dashboard

```dataview
TABLE WITHOUT ID
  file.link as "Concept",
  status as "Status",
  score as "Score",
  choice(score >= 7, "✅", choice(score > 0, "🔄", "📄")) as ""
FROM "concepts"
WHERE file.name = "review"
SORT status asc, score desc
```

## Progress

```dataviewjs
const pages = dv.pages('#ilearn/review');
const total = pages.length;
const reviewed = pages.where(p => p.status === "reviewed").length;
const inProgress = pages.where(p => p.status === "in_progress").length;
const pct = total > 0 ? Math.round(reviewed / total * 100) : 0;

dv.paragraph(`**${reviewed} / ${total}** concepts reviewed (${pct}%)`);

if (total > 0) {
  dv.paragraph(`\`\`\`
  █${'█'.repeat(Math.floor(pct/5))}${'░'.repeat(20 - Math.floor(pct/5))} ${pct}%
  \`\`\``);
}

dv.paragraph(`🔄 Needs retry: ${inProgress} | 📄 New: ${total - reviewed - inProgress}`);
```

## Weak Areas

```dataviewjs
const pages = dv.pages('#ilearn/review')
  .where(p => p.status === "reviewed" && p.breakdown);
if (pages.length > 0) {
  const avg = (key) => Math.round(pages.reduce((s, p) => s + (p.breakdown?.[key] || 0), 0) / pages.length);
  dv.table(
    ["Dimension", "Avg Score"],
    [
      ["Understanding", avg("understanding")],
      ["Depth", avg("depth")],
      ["Communication", avg("communication")]
    ]
  );
} else {
  dv.paragraph("No reviewed concepts yet.");
}
```

## Recent Interviews

```dataview
TABLE
  file.name as "Date",
  result as "Result",
  overall-score as "Score"
FROM "interviews"
WHERE file.name != ".gitkeep"
SORT file.name desc
LIMIT 10
```

> **Tip:** Install the **Dataview** plugin from Community Plugins for this dashboard to render live.
```

### Step 6: Update README.md

Append to `README.md`:

```markdown
## Obsidian Usage

1. Open this folder as an **Obsidian vault** (File → Open folder as vault).
2. Enable **Dataview** plugin from Community Plugins for the dashboard.
3. Open `obsidian-dashboard.md` to see live progress tracking.
4. Use `[[concepts/<slug>/]]` wikilinks to navigate between concepts.
5. Graph View (Ctrl/Cmd+G) shows concept relationships — filter by `#ilearn/concept`.

### Tags

- `#ilearn/concept/<slug>` — per-concept tag
- `#ilearn/status/new` — not yet reviewed
- `#ilearn/status/in_progress` — attempted but below 7/10
- `#ilearn/status/reviewed` — passed
- `#ilearn/interview` — interview logs
- `#ilearn/dashboard` — dashboards
```

### Step 7: Summary

Report to the user:

```
✅ Obsidian export complete!

Files created:
  .obsidian/app.json                    — Editor settings
  .obsidian/appearance.json             — Theme & accent color
  .obsidian/core-plugins.json           — Core plugin config
  .obsidian/community-plugins.json      — Dataview + Tracker recommended
  .obsidian/snippets/ilearn-dashboard.css  — Dashboard styling
  .obsidian/templates/concept.md        — New concept template
  .obsidian/templates/interview.md      — New interview template
  obsidian-dashboard.md                 — Live Dataview dashboard
  README.md                             — Updated with Obsidian guide

Frontmatter tags added to all concept files for graph-view filtering.

Next steps:
  1. Open this folder as a vault in Obsidian
  2. Install Dataview plugin → open obsidian-dashboard.md
  3. Use Graph View (Ctrl/Cmd+G) filtered by #ilearn/concept
```

### Step 8: Commit (if git repo)

```bash
git add .obsidian/ obsidian-dashboard.md README.md
git commit -m "feat: add Obsidian vault config and dashboard"
```

## Edge Cases

- **Already has `.obsidian/`**: Warn and ask for confirmation. Backup the old config to `.obsidian.backup/`.
- **No `review.md` for a concept**: Skip tag injection for that concept's files. Log a warning per missing file.
- **Concept already has frontmatter**: Merge tags into existing frontmatter instead of prepending. Don't double-up.
- **File has no frontmatter at all**: Prepend the frontmatter block.
- **Empty workspace (0 concepts)**: Still create `.obsidian/` config and templates, but dashboard will show empty.
- **No `.ilearn/config.json`**: Still proceed — use fallback "Unknown Topic" and "?" for levels. Print a warning.
- **Dataview not installed**: Dashboard shows raw code blocks. User needs to install Dataview plugin. Note this in the summary.
