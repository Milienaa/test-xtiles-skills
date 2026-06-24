<div align="center">

# xTiles for Claude

**Bring every conversation into your xTiles workspace.** Capture insights, plan
work, save projects, and run your day — without leaving your chat.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](.claude-plugin/plugin.json)
[![Works with Claude Code & Cowork](https://img.shields.io/badge/works%20with-Claude%20Code%20%7C%20Cowork-8A63D2.svg)](https://claude.com/claude-code)

</div>

## What is this?

[xTiles](https://xtiles.app) is a visual workspace for notes, projects, and
tasks — a flexible canvas where ideas become structured pages and plans. This
plugin connects Claude (Code & Cowork) to your xTiles workspace so Claude can
capture conversations, structure them into projects, log completed work, and run
a **Daily** planner that turns your workspace into a live morning brief —
all from natural language.

New to xTiles? Start at **[xtiles.app](https://xtiles.app)**.

## Install

### Claude Code

```
/plugin marketplace add xTiles-org/skills
/plugin install xtiles
/reload-plugins
```

### Claude Cowork

1. In the left sidebar, go to **Customize** → **Personal plugins** and click **+**
2. Click **Create plugin** → **Add marketplace**
3. In the **Add marketplace** window, paste the repository URL:
   `https://github.com/xTiles-org/skills.git`
4. Click **Sync**
5. In the **Directory** window, go to the **Personal** tab
6. Find **xtiles** in the list and click it
7. Click **Install** in the top right corner
8. Enable the **Sync automatically** toggle so you always get the latest updates

## MCP Server

The plugin ships with the xTiles MCP server preconfigured, so its tools are available the moment you install — no manual setup required.

- **Auto-connect** — installing the plugin registers the `xtiles` MCP server automatically (declared in `.mcp.json` at the plugin root). The server is remote and hosted at `https://mcp.xtiles.app/mcp`, so nothing is bundled or run locally.
- **One-time sign-in** — the first time a skill or you call an xTiles tool, Claude Code / Cowork opens your browser for a one-time OAuth login to xTiles. No tokens, API keys, or secrets are stored in the plugin — authorization is handled entirely by the host.
- **Ready to use** — after sign-in, the xTiles tools (create projects, tasks, tiles, etc.) work seamlessly alongside the skills below.

> **Upgrading from an older install?** The MCP server key was renamed from `xtiles-mcp` to `xtiles`. Re-run `/plugin install xtiles` (Claude Code) or reinstall via **Customize → Personal plugins** (Cowork) to pick up the new key — otherwise skill tool calls will fail.

See [Permissions & data](#permissions--data) for what the plugin can access, and [SECURITY.md](SECURITY.md) to report an issue.

## Skills

### Capture & Save
| Skill | Description |
|---|---|
| `/daily-note` | Save today's conversation as a note in your xTiles Planner |
| `/completed-task` | Log finished work from the current chat as a completed task |
| `/create-project` | Turn a conversation or content into a structured xTiles project — in your workspace when signed in, or as a shareable public project otherwise |

### Intelligence Hub
| Skill | Description |
|---|---|
| `/daily-brief` | Set up your Daily planner and run a personalized daily digest from connected tools — Gmail, Slack, Calendar, and more |

> The plugin also includes an internal `markdown-format` reference used by other
> skills to structure xTiles pages. It is not meant to be invoked directly.

## Examples

```
/daily-note Save today's key decisions and outcomes
/completed-task Log the API integration we just finished
/create-project Turn this research into a shareable xTiles board
/daily-brief Connect Gmail, Slack, and Calendar for a daily digest
```

### Or naturally:

```
Save this conversation to xTiles daily note
Log this as a completed task
Turn this research into a public xTiles project
```

## Permissions & data

To build your daily brief, skills can read from tools you have connected (e.g.
Gmail, Slack, Google Calendar, Notion), and they write notes, tasks, and
projects into your xTiles workspace.

- **Host-managed sign-in** — access to xTiles and any connectors goes through a
  one-time OAuth login handled by Claude Code / Cowork. **No tokens, API keys,
  or secrets are stored in the plugin.**
- **You stay in control** — skills preview what they'll create and ask before
  writing; connectors are optional and used only if you've connected them.
- **Revoke anytime** — disconnect xTiles or any connector from your host's
  plugin/connector settings.

## Contributing

Issues and pull requests are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).
To report a security or privacy issue, see [SECURITY.md](SECURITY.md).

## License

[MIT](LICENSE) © 2026 xTiles Inc.
