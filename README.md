# composio-mcp-skill

Headless Composio MCP router skill for the TrueForge agent harness.

## What it does

`composio-mcp/SKILL.md` is a skill file that teaches an MCP-based agent how to route work through Composio's MCP gateway **without any local terminal or Composio CLI** — everything happens over the MCP protocol. It encodes five non-negotiable rules:

| Rule | Mechanism |
|---|---|
| Discover tools before executing | `COMPOSIO_SEARCH_TOOLS` first; re-run on use-case change |
| Thread session state | Same `session_id` on every call in a conversation |
| Isolate users & route vault credentials | `x-user-id` routing + returned `session.mcp.headers` |
| Strict JSON type safety | Raw JSON values; never stringify primitives |
| Handle missing connections | `COMPOSIO_MANAGE_CONNECTIONS` → Connect Link → `COMPOSIO_WAIT_FOR_CONNECTIONS` |

## Install

```sh
npx skills add Gadgitmatic/composio-mcp-skill --skill composio-mcp
```

Or copy `composio-mcp/SKILL.md` into your agent's skills directory.

## Source

Extracted and sanitized from the official [Composio skills repository](https://github.com/composio-community/skills) (CLI and shell instructions removed for headless use).
