---
name: create-project
description: Convert the current chat conversation into a structured
  xTiles project. For a signed-in user the project is created in their
  workspace; otherwise a shareable public project is created. Use when
  user wants to save the chat, export conversation to xTiles, turn chat
  into a document, create a project from discussion, share conversation
  as xTiles board, structurize dialogue into tiles, save talk to xTiles.
  Also use when user provides standalone content (notes, research,
  article, brief) and wants it structured as a multi-page xTiles project.
allowed-tools: mcp__xtiles__xtiles_create_project_from_markdown
---

# Chat to xTiles Project

This skill takes the current chat (or any content the user provides) and
transforms it into a structured xTiles project with a shareable link.

Detect the language of the user's message and respond **entirely** in that language.

- Any language → same language
- **Never mix languages**
- **Never default to English**

Violating this rule is a critical failure.

## Tool and structure

- Use `mcp__xtiles__xtiles_create_project_from_markdown`
- Write Markdown strictly following the skill `markdown-format`
- The tool returns `view_id` — construct the URL as `https://xtiles.app/{view_id}`
- Plan 1–5 pages; prefer fewer, richer pages over many sparse ones

**If the user pastes additional content** alongside the chat — that content
is the primary source. The chat provides intent context only.

## Page rules

- Each page must contain minimum 4 tiles, maximum 10 tiles
- If a topic has fewer than 4 tiles — merge it with a related page
- Never create a page with 1–2 tiles; group related subtopics together
- Each page covers one distinct subtopic — no overlapping content between pages

## Typical page structures by content type

| Content type        | Suggested pages                                       |
|---------------------|-------------------------------------------------------|
| Project planning    | Overview · Goals · Tasks · Risks · Next Steps         |
| Research / article  | Summary · Key Findings · Analysis · Sources           |
| Decision discussion | Context · Options · Decision · Rationale              |
| Brainstorm          | Topic · Ideas · Shortlist · Action Items              |
| Meeting notes       | Agenda · Discussion · Decisions · Follow-ups          |

## Your process

1. **Assess the content** — how much substance is there?
2. **Plan the structure** — choose pages from the Typical page structures table; apply Page rules
3. **Prepare the Markdown** — read `skills/markdown-format/SKILL.md` and follow its rules exactly; do not paste raw chat messages
4. **Call the tool**
5. **Build the link and show the result** — see After the tool responds below

## After the tool responds

🔗 **[Share it with your team](https://xtiles.app/{view_id}), or keep building — add another page anytime by running this again in a new chat.**

CRITICAL: {view_id} must be replaced with the real `view_id` returned by the tool.
Example: `https://xtiles.app/6a180381f6c69705d68096c0`

**[Project title]**
2–3 sentences describing what the project contains.
Pages:
- [Page 1 title]
- [Page 2 title]
- [Page 3 title]

## Rules

- Always show the link first after the tool responds
- If the tool fails — show the error and suggest trying again