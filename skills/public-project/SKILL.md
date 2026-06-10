name: xtiles-public-project
description: Convert the current chat conversation into a structured public
xTiles project. Use when user wants to save the chat, export
conversation to xTiles, turn chat into a document, create a
public project from discussion, share conversation as xTiles
board, structurize dialogue into tiles, save talk to xTiles.
Also use when user provides standalone content (notes, research,
article, brief) and wants it structured as a multi-page xTiles project.
allowed-tools: mcp__xtiles__structure-information, mcp__xtiles__create-project-from-markdown
---

# Chat to xTiles Project

This skill takes the current chat (or any content the user provides) and
transforms it into a structured xTiles project with a shareable link.

## Deciding the tool and structure

**If the content is short or thin** (brief chat, simple Q&A, small topic):
- Use `mcp__xtiles__structure-information`
- Pass a well-structured prompt (see Prompt format below)
- The tool returns a ready public URL — use it directly
  **If the content is rich** (detailed chat, pasted notes, article, research):
- Use `mcp__xtiles__create-project-from-markdown`
- Write Markdown with `xtiles-markdown-format` skill
- The tool returns `view_id` — construct the URL as `https://xtiles.app/{view_id}`
- Plan 2–5 pages; do not force multi-page when single page is enough
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

## Prompt format (for structure-information)

```
Create an xTiles project titled "<title>".
 
Summary: <2–3 sentences about what this covers>
 
Content:
<Organized and enriched content>
```

## Your process

1. **Assess the content** — how much substance is there?
2. **Choose the tool** — see Deciding the tool and structure above
3. **Prepare the input** — curate and organize; do not paste raw chat messages
4. **Call the tool**
5. **Build the link and show the result** — see After the tool responds below
## After the tool responds

## After the tool responds

**If you used `structure-information`:**

🔗 Open project in xTiles: https://xtiles.app/TOOL_RESPONSE_URL

CRITICAL: ACTUAL_URL must be replaced with the real URL returned by the tool.
Example: `https://xtiles.app/6a1803637baeda338dd82052`

**[Project title]**
2–3 sentences describing what the project contains.

**If you used `create-project-from-markdown`:**

🔗 **[Share it with your team](https://xtiles.app/VIEW_ID), or keep building — add another page anytime by running this again in a new chat.**

CRITICAL: VIEW_ID must be replaced with the real `view_id` returned by the tool.
Example: `https://xtiles.app/6a180381f6c69705d68096c0`

**[Project title]**
2–3 sentences describing what the project contains.
Pages:
Page 1
Page 2
Page 3

## Rules

- Match tool and structure to content volume — see Deciding the tool above
- Curate content before passing to the tool — organize and enrich first
- Always show the link first after the tool responds
- If the tool fails — show the error and suggest trying again