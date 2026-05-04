---
name: test-xtiles-setup
description: Automatically help user fix xTiles MCP connection issues.
             ALWAYS trigger when you see any of these signals:
             "401", "unauthorized", "invalid token", "server disconnected",
             "connection refused", "MCP error", "xTiles is not connected",
             "failed to connect", "authentication failed", "Bearer",
             "mcp-remote", "XTILES_API_TOKEN".
             Do NOT wait for user to ask — trigger immediately when error appears.
allowed-tools: Bash(cat *) Bash(echo *) Bash(mkdir *) Bash(touch *)
user-invocable: false
---

# xTiles Auto Setup Guide

You detected an xTiles MCP connection error.
Stop what you were doing and show the user this message immediately:

---

❌ xTiles is not connected.

This usually means your API token is missing or expired.

⚙️ Fix in 3 steps:

**Step 1 — Get your API token**
1. Go to https://stage.xtiles.app
2. Settings → Integrations → API Tokens
3. Click "Create token" → copy it

**Step 2 — Send me your token**

Just paste the token here in the chat — I will configure everything
automatically for both Claude Code and Claude Desktop.

---

When the user sends the token, detect the environment:

### Claude Code (terminal)

Read current config:    cat ~/.claude/settings.json   Merge and write back mcpServers to ~/.claude/settings.json:
```json
{
  "mcpServers": {
    "xtiles": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp-stage.xtiles.app/mcp",
        "--header",
        "Authorization: Bearer <TOKEN_FROM_USER>"
      ]
    }
  }
}
```

### Claude Desktop / Cowork (macOS)

Read current config:   cat ~/Library/Application\ Support/Claude/claude_desktop_config.json
   Merge and write back mcpServers to the same file:
```json
{
  "mcpServers": {
    "xtiles": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp-stage.xtiles.app/mcp",
        "--header",
        "Authorization: Bearer <TOKEN_FROM_USER>"
      ]
    }
  }
}
```

### Claude Desktop / Cowork (Windows)

Read current config:   cat $APPDATA\Claude\claude_desktop_config.json
  
Merge and write back mcpServers to the same file.

---

After writing the config show:

✅ Token configured successfully!

**Step 3 — Restart to apply**

- Claude Code → run /reload-plugins
- Claude Desktop → restart the app

I will retry your original request automatically after restart.

---

**Common errors:**
❌ 401 Unauthorized → token is wrong or expired → repeat Step 1
❌ Server disconnected → config file not found or wrong path → repeat Step 2
❌ npx not found → install Node.js from https://nodejs.org