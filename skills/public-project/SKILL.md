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

Some conversations are too valuable to stay in a chat window. This skill
takes the current chat and transforms it into a structured public xTiles
project — a visual, shareable document that captures the full context,
ideas, and outcomes of the discussion.

## What this skill does

Reads the current chat, structures it into a titled and summarized
document, and publishes it as a public xTiles project. Returns a
shareable link that anyone can open — no xTiles account needed.

## The result the user gets

A public xTiles project that contains:
- A meaningful title derived from the chat topic
- A short summary of what the conversation covers
- The full dialogue structured into visual tiles
- A shareable URL ready to send or publish

## Your process

1. **Collect the conversation** — gather the full chat history from context
2. **Derive a title** — pick a meaningful name based on the main topic
3. **Write a summary** — 1-3 sentences describing what the chat covers
4. **Format the input** — assemble title, summary, and dialogue into one text
5. **Call the tool** — pass the prepared text to mcp__xtiles__structure-information
6. **Return the link** — extract the URL from the tool response and show it
   to the user as the first and most prominent element of the reply

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

## Rules

- Include both user and assistant messages — full context matters
- Derive a meaningful title from the topic, never use "Chat export"
- If chat is long (50+ messages) — summarize older parts, keep last 20 verbatim
- Call the tool immediately — no confirmation needed from the user
- Always surface the URL as the first thing shown after the tool responds
- If the tool fails — show the error and suggest trying again