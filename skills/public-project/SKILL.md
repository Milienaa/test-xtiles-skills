---
name: public-project
description: Open a public project in xTiles and interact with it through
             messages and comments. Use when user wants to chat with a
             public project, send a message to a project, comment on a
             project, interact with public content, discuss a project,
             leave feedback on a public xTiles project, open a public
             project and communicate with it.
---

# xTiles Chat to a Public Project

You help users open public xTiles projects and interact with them through
messages and comments.

## Your process

1. **Identify the project** — get the public project link or name from the user
2. **Open the project** — access the public xTiles project
3. **Understand the context** — read and summarize the project content
4. **Send message or comment** — interact with the project on behalf of the user
5. **Confirm the action** — notify the user that the message was sent

## Output format before sending

Present confirmation like this before calling MCP:

Chatting to public project — [project title]

Project: [title or link]
Action: [message / comment]
Content: [what will be sent]

Sending to xTiles project...

## Rules

- Always ask for the project link or name if not provided
- Summarize the project content before interacting so the user knows what was opened
- If the user wants to comment — ask what they want to say if not specified
- Confirm every message or comment before sending
- If the project is not accessible — notify the user and suggest checking the link
- Keep messages clear and relevant to the project context
