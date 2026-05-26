---
name: xtiles-public-project
description: Convert the current chat conversation into a structured public
   xTiles project. Use when user wants to save the chat, export
   conversation to xTiles, turn chat into a document, create a
   public project from discussion, share conversation as xTiles
   board, structurize dialogue into tiles, save talk to xTiles.
   Also use when user provides standalone content (notes, research,
   article, brief) and wants it structured as a multi-page xTiles project.
allowed-tools: mcp__xtiles__structure-information, mcp__xtiles__create_project_from_markdown
---

# Chat to xTiles Project

This skill takes the current chat (or any content the user provides) and
transforms it into a structured xTiles project with a shareable link.

## How the tool works — CRITICAL

`mcp__xtiles__create_project_from_markdown` uses Markdown heading levels
to define structure:

- `#` — project title (exactly one, at the top)
- `##` — each heading becomes a separate page (view)
- `###` — each heading becomes a tile inside the page above it

## Deciding the structure

**If the content is rich** (detailed chat, pasted notes, article, research)
— create a multi-page project with 2–5 pages. Use the typical structures
below as a guide.

**If the content is short or thin** (brief chat, simple question/answer,
small topic) — create a single-page project with well-organized tiles.
Do not force multi-page structure where it is not needed.

**If the user pastes additional content** alongside the chat — that content
is the primary source. The chat provides intent context only.

## Typical page structures by content type

| Content type        | Suggested pages                                       |
|---------------------|-------------------------------------------------------|
| Project planning    | Overview · Goals · Tasks · Risks · Next Steps         |
| Research / article  | Summary · Key Findings · Analysis · Sources           |
| Decision discussion | Context · Options · Decision · Rationale              |
| Brainstorm          | Topic · Ideas · Shortlist · Action Items              |
| Meeting notes       | Agenda · Discussion · Decisions · Follow-ups          |

## Markdown format for the tool

Multi-page:
```
# <Project title>

## <Page 1 name>

### <Tile title>
<Tile content>

### <Tile title>
<Tile content>

## <Page 2 name>

### <Tile title>
<Tile content>
```

Single-page:
```
# <Project title>

## <Page name>

### <Tile title>
<Tile content>

### <Tile title>
<Tile content>
```

## Your process

1. **Assess the content** — how much substance is there?
2. **Choose the structure** — multi-page for rich content, single-page for thin
3. **Write the Markdown** — curate and organize; do not paste raw chat messages
4. **Call the tool** — pass the Markdown to `mcp__xtiles__create-project-from_markdown`
5. **Build the URL** — the tool returns `project_id` and `view_id`; construct
   the link as `https://app.xtiles.app/{view_id}`
6. **Show the result**

## After the tool responds

Show this as the first thing in your reply:

🔗 **[Open project in xTiles](https://app.xtiles.app/{view_id})**

Then: project title and one-sentence description.
For multi-page projects also list the pages: Page 1 · Page 2 · Page 3

## Rules

- Always use `#` / `##` / `###` hierarchy — this is what creates pages and tiles
- Match structure to content — don't force multi-page when single-page is enough
- Curate content — organize and enrich before passing to the tool
- Always show the link first after the tool responds
- If the tool fails — show the error and suggest trying again