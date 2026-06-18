# Contributing to xTiles for Claude

Thanks for your interest in improving the xTiles plugin! This repository packages
a set of **skills** plus a preconfigured MCP server for Claude Code and Claude
Cowork.

## Repository layout

| Path | What it is |
|------|------------|
| `.claude-plugin/plugin.json` | Plugin manifest (name, version, skills path, license) |
| `.claude-plugin/marketplace.json` | Marketplace entry used by `/plugin marketplace add` |
| `.mcp.json` | Preconfigured remote MCP server (`xtiles`) |
| `skills/<name>/SKILL.md` | One folder per skill |
| `README.md` | Overview & install |

## Adding or editing a skill

Each skill is a single `SKILL.md` with YAML frontmatter:

```yaml
---
name: my-skill                 # kebab-case, matches the folder name
description: >                 # when the skill should trigger — be specific
  One or two sentences describing when to use this skill, with example
  trigger phrases.
allowed-tools: mcp__xtiles__xtiles_create_tasks, Read, Write, AskUserQuestion
---
```

Guidelines:

- **Tool names must match the real MCP tools exactly.** The pattern is
  `mcp__<server-key>__<tool-name>`. The server key is `xtiles` (from `.mcp.json`)
  and every xTiles tool name starts with `xtiles_`, so the full identifier looks
  like `mcp__xtiles__xtiles_create_tasks` — note the double `xtiles` is correct,
  not a typo. Never use hyphens or omit the server prefix —
  e.g. `mcp__xtiles__create-tasks` or `mcp__create_tasks` will fail silently.
  Verify each name against the live tool list before submitting.
- Keep the body focused: a clear step-by-step process, the exact tool calls,
  and the output/confirmation format.
- **Preview before writing** to a user's workspace, and never invent data
  (names, events, messages) — use only what the tools return.
- Match the language of the user's conversation.

## Testing locally

1. Add the marketplace from your local checkout and install the plugin in
   Claude Code / Cowork.
2. Run the skill (for example `/daily-note`) and confirm the tool calls resolve
   and the output is correct.

## Pull requests

- One logical change per PR; describe **what** changed and **why**.
- Keep `plugin.json` and `marketplace.json` valid JSON.
- Be kind and constructive.
