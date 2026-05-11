---
name: xtiles-public-project
description: Convert the current chat conversation into a structured public
             xTiles project. Use when user wants to save the chat, export
             conversation to xTiles, turn chat into a document, create a
             public project from discussion, share conversation as xTiles
             board, structurize dialogue into tiles, save talk to xTiles.
allowed-tools: mcp__xtiles__structure-information
---

# Chat to xTiles Public Project

You convert the current chat conversation into a well-structured public
xTiles project and return a shareable link to the user.

## Your process

1. **Collect the conversation** — gather the full chat history from context
2. **Derive a title** — pick a meaningful name based on the main topic
3. **Write a summary** — 1-3 sentences describing what the chat covers
4. **Format the input** — assemble title, summary, and dialogue into one text
5. **Call the tool** — pass the prepared text to mcp__xtiles__structure-information
6. **Return the link** — present the shareable URL to the user

## Input format for the tool

Prepare the text like this before calling MCP:

# [Title derived from chat topic]

## Summary
[1-3 sentences about what this conversation covers]

## Conversation

User: [message]
Assistant: [reply]

User: [message]
Assistant: [reply]

## Output after saving

Present the result like this after the tool responds:

🗂️ xTiles project created!

📎 Public link: [URL returned by the tool]

Topic: [derived title]

Open the link to view and share your structured project.

## Rules

- Include both user and assistant messages — full context matters
- Derive a meaningful title from the topic, never use "Chat export"
- If chat is long (50+ messages) — summarize older parts, keep last 20 verbatim
- Call the tool immediately — no confirmation needed from the user
- Always surface the URL prominently in the response
- If the tool fails — show the error and suggest trying again