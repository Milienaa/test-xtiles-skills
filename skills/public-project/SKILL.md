---
name: xtiles-public-project
description: Convert the current chat conversation into a structured public
   xTiles project. Use when user wants to save the chat, export
   conversation to xTiles, turn chat into a document, create a
   public project from discussion, share conversation as xTiles
   board, structurize dialogue into tiles, save talk to xTiles.
   Also use when user provides standalone content (notes, research,
   article, brief) and wants it structured as a multi-page xTiles project.
allowed-tools: mcp__xtiles__structure-information
---

# Chat to xTiles Public Project

Some conversations are too valuable to stay in a chat window. This skill
takes the current chat (or standalone content) and transforms it into a
structured public xTiles project — a visual, shareable document that
captures the full context, ideas, and outcomes of the discussion.

## Two modes

### Mode A — Chat export (default)
The user runs the skill in an existing conversation without extra content.
The full chat is the source material.

### Mode B — Content structuring (preferred quality)
The user pastes notes, an article, a brief, research, or any rich content.
That content is the primary source; the chat provides intent context only.

Mode B produces significantly higher quality output. If the chat is short
or thin, suggest it:
> "The current chat is brief. Paste your notes or content for a richer
> project, or reply 'export chat' to proceed as-is."

## The result the user gets

A public xTiles project with:
- A meaningful title derived from the topic
- A short summary of what the project covers
- Content structured into 2–5 logical pages, each with a clear focus
- A shareable URL ready to send or publish

## Content structuring principles

Plan the page structure before calling the tool:

- Do not put everything on one page — identify 2–5 logical sections
- Each page should have a single clear focus
- Enrich the content: expand thin points, surface implicit conclusions explicitly
- For chat exports: extract signal from noise — focus on decisions and outcomes,
  ignore small talk

## Typical page structures by content type

| Content type        | Suggested pages                               |
|---------------------|-----------------------------------------------|
| Project planning    | Overview · Goals · Tasks · Risks · Next Steps |
| Research / article  | Summary · Key Findings · Analysis · Sources   |
| Decision discussion | Context · Options · Decision · Rationale      |
| Brainstorm          | Topic · Ideas · Shortlist · Action Items      |
| Meeting notes       | Agenda · Discussion · Decisions · Follow-ups  |

## Your process

1. **Detect mode** — is there rich pasted content, or just a chat?
2. **If chat is thin** (fewer than 10 substantive exchanges) — suggest Mode B
3. **Plan the structure** — decide on 2–5 pages and their focus areas
4. **Format the input** — assemble structured text using the Input format below
5. **Call the tool** — pass the prepared text to `mcp__xtiles__structure-information`
6. **Return the link** — show the URL as the first element of the reply


## Input format for the tool

Prepare a rich, well-structured prompt that tells the xTiles AI exactly
what kind of project to create:

```
Create a multi-page xTiles project titled "<Title>".
 
## Project summary
<2–3 sentences about what this project covers and who it is for>
 
## Page structure
This project should have the following pages:
1. <Page name> — <what it contains>
2. <Page name> — <what it contains>
3. <Page name> — <what it contains>
 
## Content
 
### <Page 1 name>
<Structured content for this page>
 
### <Page 2 name>
<Structured content for this page>
 
### <Page 3 name>
<Structured content for this page>
```

The richer and more structured this input, the better the xTiles output.
Do not just paste raw chat messages — curate, organize, and enrich the content
before passing it to the tool.

## After the tool responds

Show the result like this:
 
---
🔗 Open project in xTiles: {URL returned by the tool}

Title: {project title} — one-sentence description
Pages: {page 1} · {page 2} · {page 3}
---

The URL is always unique per project — extract it from the tool response
and render it as a real clickable link in the output.

## Rules

- Always surface the URL as the first thing shown after the tool responds
- Derive a meaningful title — never use "Chat export" or "Conversation"
- Plan multi-page structure before calling the tool — single-page output is a failure
- For thin chats: ask for more content rather than producing low-quality output
- If the tool fails — show the error and suggest trying again
- Include both user and assistant messages in chat exports (curated, not raw)