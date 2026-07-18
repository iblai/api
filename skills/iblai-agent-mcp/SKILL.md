---
name: iblai-agent-mcp
description: Manage an ibl.ai agent's MCP connectors via the platform API — list connectors, create/edit/delete them, enable/disable on an agent, and complete OAuth connections to connected services. Use when wiring an agent to external MCP servers and tools.
---

# iblai-agent-mcp

Manage an agent's MCP connectors through the API: browse and manage MCP
connectors, enable/disable them on an agent, and complete OAuth connections to
external services. Use when wiring an agent to external MCP servers and tools.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `…/users/{username}/mcp-servers/?include_global=true&mentor_unique_id={mentor}&is_featured={true|false}&page={n}&page_size=12&search={q}&transport={…}` — connector lists.
- **GET** `…/mentors/{mentor}/settings/` — active `mcp_servers`.
- **GET** `…/users/{username}/mcp-server-connections/` — OAuth connection state.
- **GET** `https://api.iblai.app/dm/api/ai-account/connected-services/orgs/{org}/users/{username}/` — connected OAuth services.
- **GET** `…/connected-services/orgs/{org}/users/{username}/{provider}/{service}/` — start OAuth (returns `auth_url`).

## Writes

- **POST** `…/users/{username}/mcp-servers/` — create a connector (JSON, or `multipart/form-data` with image):
  ```json
  {
    "name": "string (required)",
    "url": "string (required)",
    "transport": "sse|websocket|streamable_http (required)",
    "auth_type": "none|token|oauth2 (required)",
    "description": "string",
    "credentials": "string",
    "auth_scope": "user|mentor|tenant",
    "mentor": "uuid|null",
    "image": "File"
  }
  ```
- **PUT** `…/mcp-servers/{id}/` — edit a connector.
- **DELETE** `…/mcp-servers/{id}/` — delete a connector (no body). Destructive — confirm with the user first.
- **PUT** `…/mentors/{mentor}/settings/` — enable / disable a connector on the agent:
  ```json
  {
    "mcp_servers": "number[]",
    "tool_slugs": "string[]",
    "can_use_tools": "boolean"
  }
  ```
- **POST** `…/users/{username}/mcp-server-connections/` — finalize an OAuth connection:
  ```json
  {
    "server": "number (required)",
    "scope": "user|mentor|tenant (required)",
    "auth_type": "oauth2 (required)",
    "connected_service": "number (required)",
    "user": "string",
    "mentor": "uuid"
  }
  ```
- **DELETE** `…/connected-services/orgs/{org}/users/{username}/{id}/` — disconnect an OAuth service (no body). Destructive — confirm with the user first.

## Example

Create an SSE connector with no auth and enable it on the agent:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mcp-servers/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Weather MCP",
    "url": "https://mcp.example.com/sse",
    "transport": "sse",
    "auth_type": "none",
    "mentor": "'"$MENTOR"'"
  }'
```

## Connector model

Three objects work together:

- **MCP server** (`mcp-servers/`) — the connector record: `name`, `url`,
  `transport` (`sse|websocket|streamable_http`), `auth_type`
  (`none|token|oauth2`), and `auth_scope` (`user|mentor|tenant`).
  `is_featured=true` lets admins in *other* orgs create their own connections to
  the same server; disabled connectors are skipped at runtime.
- **MCP server connection** (`mcp-server-connections/`) — the auth binding that
  supplies credentials for a server at a given `scope`. OAuth2 connections
  reference a `connected_service`.
- **Connected service** (`connected-services/…`) — a user's stored OAuth grant
  for a provider + service (e.g. `google` + `drive`), created by completing the
  OAuth flow. One per user per service.

`auth_type` and `auth_scope` answer two different questions: `auth_type` is
*how* the call is authenticated (`none`, a static `token`, or `oauth2`);
`auth_scope`/`scope` is *whose* credentials are used (`user`, `mentor`, or
`tenant`). Only `auth_type=oauth2` **and** `auth_scope=user` together trigger the
in-chat OAuth prompt below.

## Runtime credential resolution

When an agent calls an attached MCP server, the platform selects credentials in
priority order — first match wins:

1. **user**-scoped connection for (server, current user)
2. **mentor**-scoped connection for (server, agent)
3. **tenant**-scoped connection for (server, org)
4. featured-server global fallback
5. otherwise fail (401 / no connection) — or, for a per-user oauth2 server with
   no user connection, start the in-chat OAuth flow

OAuth access tokens are refreshed automatically as they near expiry — no client
action required.

## In-chat OAuth (per-user oauth2 connectors)

A connector with `auth_type=oauth2` + `auth_scope=user` authenticates each user
individually. When an authenticated user chats with the agent and has no
connection yet, the chat pauses and the runtime emits JSON events on the
existing chat socket (WebSocket/SSE) — switch on the `type` field:

- `oauth_required` — carries `server_name`, `server_id`, and `auth_url`; open
  `auth_url` in a new tab/popup (providers block iframes) so the user can
  authenticate. Backend then polls for the new connection.
- `oauth_connection_resolved` — the connection was created; dismiss the prompt,
  the chat resumes automatically.
- `warning` (`code` 503) — MCP tools couldn't load for a non-OAuth reason
  (server unreachable / misconfigured); chat continues without those tools.
- `error` (`status_code` 400) — OAuth timed out (default ~5 min), the auth URL
  couldn't be built (missing `auth_{provider}` credential), or the connection
  has no valid connected service; the user retries the message.

The backend polls every ~10s and times out after ~300s. Completing OAuth *after*
a timeout still works — the next message succeeds because the connection now
exists.

## Notes

- The connector list is paged (`page_size=12`); use `search` and `transport` to
  narrow results, and `include_global=true` to surface org-wide connectors.
- Enabling a connector is a toggle through `settings/` — send the full
  `mcp_servers` array of connector ids you want active on the agent.
- OAuth is two steps: **GET** the connected-services start endpoint for an
  `auth_url`, let the user complete it, then **POST** `mcp-server-connections/`
  with the resulting `connected_service`.
- `auth_scope` / `scope` (`user|mentor|tenant`) controls who the connection
  applies to — `user` is per-user, `mentor` is agent-wide, `tenant` is org-wide.
