# xTiles Skills

Bring every conversation into your xTiles workspace. Capture insights, plan work, save projects, and stay on top of your day — without leaving your chat.

## Install

### Claude Code

```
/plugin marketplace add xTiles-org/skills
/plugin install productivity
/reload-plugins
```

### Claude Cowork

1. In the left sidebar, go to **Customize** → **Personal plugins** and click **+**
2. Click **Create plugin** → **Add marketplace**
3. In the **Add marketplace** window, paste the repository URL:
   `https://github.com/xTiles-org/skills.git`
4. Click **Sync**
5. In the **Directory** window, go to the **Personal** tab
6. Find **productivity** in the list and click it
7. Click **Install** in the top right corner
8. Enable the **Sync automatically** toggle so you always get the latest updates

## MCP Server

The plugin ships with the xTiles MCP server preconfigured, so its tools are available the moment you install — no manual setup required.

- **Auto-connect** — installing the plugin registers the `xtiles-mcp` server automatically (declared in `.mcp.json` at the plugin root). The server is remote and hosted at `https://mcp.xtiles.app/mcp`, so nothing is bundled or run locally.
- **One-time sign-in** — the first time a skill or you call an xTiles tool, Claude Code / Cowork opens your browser for a one-time OAuth login to xTiles. No tokens, API keys, or secrets are stored in the plugin — authorization is handled entirely by the host.
- **Ready to use** — after sign-in, the xTiles tools (create projects, tasks, tiles, etc.) work seamlessly alongside the skills above.

## Skills

### Capture & Save
| Skill | Description |
|---|---|
| `/daily-note` | Save today's conversation as a note in your xTiles Planner |
| `/completed-task` | Log finished work from the current chat as a completed task |
| `/public-project` | Turn a conversation or content into a structured public xTiles project |

### Intelligence Hub
| Skill | Description |
|---|---|
| `/intelligence-hub-digest` | Set up your cascading Month → Week → Day planner and run a personalized daily digest from connected tools — Gmail, Slack, Calendar, Notion, and more |

## Examples

```
/daily-note Save today's key decisions and outcomes
/completed-task Log the API integration we just finished
/public-project Turn this research into a shareable xTiles board
/intelligence-hub-digest Connect Gmail, Slack, and Calendar for a daily digest
```

## Or naturally:

```
Save this conversation to xTiles daily note
Log this as a completed task
Turn this research into a public xTiles project
```
