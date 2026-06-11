---
name: mcp-setup
description: >
  Help user connect xTiles MCP on their own machine.
  Trigger when user asks how to set up xTiles, connect MCP, or configure token.
  Also trigger immediately on any of these errors without waiting for user to ask:
  401, unauthorized, invalid token, server disconnected, MCP error,
  xTiles is not connected, failed to connect, authentication failed,
  Bearer, mcp-remote, XTILES_API_TOKEN.
user-invocable: false
---

# xTiles MCP Setup Guide

To use xTiles tools you need to configure the xTiles MCP server.
Show the user this message:

---

## Connect xTiles MCP

### Claude Cowork

#### Step 1 — Open Customize
Go to **Customize → Connectors**.

#### Step 2 — Add a custom connector
Click **Add custom connector**, name it `xTiles`, and paste the URL:
```
https://mcp.xtiles.app/mcp
```

#### Step 3 — Connect and authorize
Click **Add → Connect**, sign in to your xTiles account, then click **Allow access** — you're all set.

---

### Claude Code

#### Step 1 — Get your API token
1. Go to https://xtiles.app
2. Settings → Integrations → API Tokens
3. Click "Create token" → copy it (starts with `xt_`)

#### Step 2 — Add MCP config

**Claude Desktop**
1. Open Claude Desktop → **Settings → Developer** (bottom of the sidebar)
2. Click **Edit Config** — it opens the folder with the config file
3. Open `claude_desktop_config.json` in any text editor
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
4. Add the block below. If the file is empty or has only `{}`, paste the full config.
   If other servers already exist — add a new entry separated by a comma.

```json
{
  "mcpServers": {
    "xtiles": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.xtiles.app/mcp",
        "--header",
        "Authorization: Bearer YOUR_TOKEN_HERE"
      ]
    }
  }
}
```

Replace `YOUR_TOKEN_HERE` with the token from Step 1.

**Claude Code**
Add the same `mcpServers` block to `~/.claude/settings.json`.

#### Step 3 — Restart to apply
- **Claude Desktop** → quit and reopen the app
- **Claude Code** → run `/reload-plugins`

---

**Common errors:**
- 401 Unauthorized → token wrong or expired → repeat Step 1
- Server disconnected → config path wrong → check Step 2
- `npx` not found → install Node.js from https://nodejs.org