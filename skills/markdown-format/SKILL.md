---
name: markdown-format
description: >
  Reference guide for xTiles Markdown structure used by xtiles_create_project_from_markdown.
  Not user-invocable — used internally only by the create-project skill.
user-invocable: false
---

# xTiles Markdown Format Reference

This document describes how `mcp__xtiles__xtiles_create_project_from_markdown` interprets
Markdown heading levels to build project structure.

**Scope — only for `xtiles_create_project_from_markdown`.** This file backs the
`create-project` skill exclusively. The `reorganize` / `reorganize-with-gpt` skills
call a different tool, `xtiles_create_view_from_markdown`, and define their own
grid, sizing, and color rules inline in their own SKILL.md — including tile
heights well past `h=12` for content-heavy tiles. Never apply this file's fixed
`h=12` / allowed-widths rule to those skills' output.

## Conversion to MD Structure Rules

---

### Overall Structure

- Start with `# [Project Name]`
- Then `## [Theme]` — the view for this topic
- Add cover: `@cover: [keyword]`
- Structure content into `### Tile` sections

---

### Tile Rules

- Each tile needs a meaningful title with emoji
- Metadata goes right after the title (no blank line between title and metadata):
  - `@position: x, y, w, h`
  - `@colorSize: [style from available]`
  - `@color: [color]`
- One empty line after metadata before content
- NO subheadings (`####`) inside tiles — use **bold** instead

---

### Tile Content Rules

- `- [ ]` task items for actionable steps — one empty line between each
- `-` bullet points for lists and ideas
- **Bold text** for emphasis and section headers within tiles
- `>` quotes for motivational phrases
- NO nested lists

---

### Canvas & Positioning

- Grid is 48 units wide
- All tiles: `h=12` (fixed height)
- Allowed widths: `16`, `24`, `32`, `48`
- Each row must sum to exactly 48:

| Combination     |
|-----------------|
| 48              |
| 32 + 16         |
| 24 + 24         |
| 24 + 16 + 16    |
| 16 + 16 + 16    |

---

### Styling Rules

- Analyze the conversation theme
- Select two complementary colors from the list below
- Use ONLY those two colors for all tiles
- Alternate them for visual variety

---

### Available Colors
GHOST, CUMULUS, GOSSIP, COLDTURKEY, BLUE_CHALK, MILK_PUNCH,
HAWKES_BLUE, PATTENS_BLUE, SAIL, ATHENS_GRAY, BERMUDA,
PERFUME, SELAGO, RICE_FLOWER, WHITE_LINEN, POLAR

### Available Styles
LIGHTER, HEADER, LIGHTER_HEADER, LIGHTER_CONTOUR_LINE_BORDER

