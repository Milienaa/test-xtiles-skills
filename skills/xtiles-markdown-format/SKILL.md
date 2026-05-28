---
name: xtiles-markdown-format
description: >
  Reference guide for xTiles Markdown structure used by create_project_from_markdown.
  Not user-invocable — used internally by other skills.
user-invocable: false
---

# xTiles Markdown Format Reference

This document describes how `mcp__xtiles__create-project-from-markdown` interprets
Markdown heading levels to build project structure.

## Heading hierarchy

- `#` — project title (exactly one, at the very top)
- `##` — each heading becomes a separate page (view) inside the project
- `###` — each heading becomes a tile inside the page above it
- Body text under `###` becomes the tile content
## Example

```markdown
# Project Title
 
## Page One
 
### Tile A
Content for tile A.
 
### Tile B
Content for tile B.
 
## Page Two
 
### Tile C
Content for tile C.
```

This produces one project with two pages: "Page One" (tiles A, B) and "Page Two" (tile C).

## Rules

- Always start with exactly one `#` — the project title
- Always have at least one `##` — at least one page is required
- Every `###` must be under a `##` — never use `###` at the top level
- Do not skip levels — never go from `#` directly to `###`
- Body text outside of `###` tiles is ignored or treated as page description