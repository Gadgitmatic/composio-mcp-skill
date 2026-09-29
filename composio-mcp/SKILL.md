---
name: composio-mcp
description: MCP gateway routing rules for Composio. Handles dynamic tool search, session_id threading, user isolation headers, and JSON type safety for headless harnesses.
---

# Composio MCP Gateway Router

Headless MCP routing rules for Composio. This skill assumes no local terminal, no Composio CLI, and no shell access — all Composio interaction happens exclusively through the MCP gateway.

## 0. Endpoint & Auth — Connect vs Project Keys

Two ways to reach Composio over MCP. Prefer **Connect** for ease of setup.

**Composio Connect (recommended).** Hosted MCP server at `https://connect.composio.dev/mcp`
(streamable HTTP). Exposes the `COMPOSIO_*` meta-tools used below. Authenticate with a
**consumer key** (`ck_…`) in the `x-consumer-api-key` header — **not** your project API
key (`ak_…`); the Connect endpoint does not accept project keys. Get the consumer key in
the Composio dashboard under **For You → Connect Settings → Sessions & API Key**.

```json
{
  "mcpServers": {
    "composio": {
      "type": "http",
      "url": "https://connect.composio.dev/mcp",
      "headers": { "x-consumer-api-key": "ck_…" }
    }
  }
}
```

The first time the agent needs an app, Composio returns an OAuth link: the user approves
it in the browser and the connection persists across sessions. If the MCP host itself
authenticates via OAuth (the client holds the grant), no API key is needed at all and
the agent never sees credential headers.

**Project MCP server (scoped alternative).**
`https://backend.composio.dev/v3/mcp/<server_id>?user_id=<user_id>` with header
`x-api-key: ak_…`. Use when you need per-toolkit servers or explicit user scoping.
Create the server in the Composio dashboard (or SDK), then connect with the generated URL.

## When to Apply

- The harness (TrueForge) must access external apps (Gmail, Slack, GitHub, Notion, etc.) via Composio's MCP server.
- A task requires discovering tools at runtime instead of loading fixed tool schemas.
- Actions must run on behalf of a specific isolated user.
- A tool call fails because the user has no active connection to the target app.

## Non-Negotiable Rules

1. **Search before executing.** Always call `COMPOSIO_SEARCH_TOOLS` first. Never invent tool slugs.
2. **Thread the session.** Pass the same `session_id` on every call in a conversation.
3. **Isolate users.** Route identity via the `x-user-id` header. Never mix users on one session.
4. **Respect JSON types.** Arguments are JSON values, never stringified primitives.
5. **Handle missing connections.** Redirect through `COMPOSIO_MANAGE_CONNECTIONS` and the Connect Link flow.

---

## 1. Tool Discovery — `COMPOSIO_SEARCH_TOOLS`

Call `COMPOSIO_SEARCH_TOOLS` first, at the start of any task that touches an external app or service. Re-run it whenever the user pivots to a new use case.

- The response returns matching tool slugs, their input schemas (or references fetchable via `COMPOSIO_GET_TOOL_SCHEMAS`), per-toolkit **connection status**, and a suggested execution plan/use-case playbook.
- Execute only tools returned by search. Using a hallucinated or stale tool slug causes a runtime error.
- If the search response includes connection status, check `has_active_connection` for the target toolkit **before** executing.
- Do not rely on a previous conversation's tool list — re-search when the task changes so discovery state stays within the current session.

## 2. Session ID Threading

Composio sessions are stateful: discovery results, connection status, and workbench state live inside the session.

- On first contact, a session is created and a `session_id` is returned. Persist it.
- Pass the **same** `session_id` with every subsequent call (meta-tool calls and regular tool executions alike) so search results, connection state, and execution context stay consistent.
- Never create a new session per call within the same conversation; that resets discovery state and can orphan in-progress connection flows.
- Multi-turn: create one session per user conversation, store `session_id` alongside the conversation, and reuse it across turns.

```
turn 1: session created -> session_id = "sess_abc123"  (store it)
turn n: every request carries session_id = "sess_abc123"
```

## 3. Identity & Vault Routing — `x-user-id`

Credentials (OAuth tokens, API keys) are stored per user in Composio's vault and injected at execution time. They never pass through your application or the model.

- Route every request with the correct `x-user-id` header so tool executions run against **that user's** connected accounts. User A's session must never touch user B's data.
- Use a stable, non-changing identifier (e.g., your database ID) as the user ID — never an email or mutable username.
- `session.mcp.headers` carries the auth credential headers (`x-api-key` or `x-user-api-key`, plus `x-org-id` / `x-project-id` when scoped). Pass them through **exactly as returned** — do not assemble your own headers from environment variables. Treat them as secrets.
- Never forward the session headers to a different origin than the MCP URL they were issued for.

## 4. Strict JSON Primitive Safety

Tool arguments are JSON **values**, not strings. Stringifying primitives is the most common headless failure mode.

- ❌ `{"subject": "Hello", "max_results": "10", "pinned": "true", "labels": "[\"work\"]"}`
- ✅ `{"subject": "Hello", "max_results": 10, "pinned": true, "labels": ["work", "urgent"]}`

| Type | Handling |
|---|---|
| Integer / float | Pass raw numeric literal — not `"10"` |
| Boolean | Pass `true` / `false` — not `"true"` |
| Array / object | Pass a real JSON array / object — never a JSON-encoded string |
| Long content | Keep raw text inline; do not pre-escape or wrap in extra quotes |

If the model serializes by stringifying the entire arguments object, unwrap it before sending so each field retains its native JSON type.

## 5. Connection State Fallbacks — `COMPOSIO_MANAGE_CONNECTIONS`

When `COMPOSIO_SEARCH_TOOLS` reports `has_active_connection: false` (or any call later fails with an auth/connection error), execution is blocked until the toolkit is connected:

1. Call **`COMPOSIO_MANAGE_CONNECTIONS`** for the toolkit. It returns a secure **Connect Link** where credentials are created and stored in the vault — the user's browser handles OAuth/API-key entry; the harness never sees credentials.
2. Present the Connect Link to the user with a clear ask: "Connect `gmail` to continue: https://connect.composio.dev/link/…". Surfacing the link in chat is safe by design.
3. Wait for completion via **`COMPOSIO_WAIT_FOR_CONNECTIONS`** (or a redirect flow with a `callback_url` back to the harness, if configured).
4. Retry the original tool only after connection status is `ACTIVE`.
5. OAuth links are short-lived — if one expires, regenerate via `COMPOSIO_MANAGE_CONNECTIONS` rather than telling the user to retry blindly.

Failure loop to avoid: calling the tool repeatedly while disconnected. Always route through the connection flow first.

---

## Quick Reference

| Rule | Meta-tool / mechanism |
|---|---|
| Discover tools | `COMPOSIO_SEARCH_TOOLS` (call first; re-run on use-case change) |
| Fetch schemas | `COMPOSIO_GET_TOOL_SCHEMAS` |
| Execute | `COMPOSIO_MULTI_EXECUTE_TOOL` (up to 50 parallel calls) |
| Connect app | `COMPOSIO_MANAGE_CONNECTIONS` → Connect Link |
| Await auth | `COMPOSIO_WAIT_FOR_CONNECTIONS` |
| Thread state | `session_id` on every call |
| User isolation | `x-user-id` routing + returned `session.mcp.headers` |
| Type safety | Raw JSON values; never stringify primitives |
