# Security Policy

## Reporting a vulnerability

If you discover a security or privacy issue in this plugin, please report it
**privately** — do **not** open a public GitHub issue.

- Email: **support@xtiles.app**
- Or use GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)
  on this repository.

We aim to acknowledge reports within a few business days and will coordinate a
fix and a disclosure timeline with you.

## Supported versions

| Version | Supported |
|---------|-----------|
| 1.0.x   | ✅        |

## What this plugin can access

- The plugin ships a preconfigured **remote** MCP server
  (`https://mcp.xtiles.app/mcp`). Nothing is bundled or run locally.
- Access to xTiles — and to any optional connectors you enable (Gmail, Slack,
  Google Calendar, Notion, …) — is authorized through a **one-time OAuth
  sign-in handled by the host** (Claude Code / Cowork).
- **No tokens, API keys, or secrets are stored in the plugin.** Authorization is
  managed entirely by the host.
- Skills request only the data needed for the action you invoked, preview what
  they will create, and ask before writing to your workspace. You can revoke
  access at any time from your host's plugin/connector settings.
