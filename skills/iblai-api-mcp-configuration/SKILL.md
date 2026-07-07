---
name: ibl-mcp-config
description: >-
  Configure MCP (Model Context Protocol) servers, connections, and OAuth
  connectors on the ibl.ai platform via its REST API. Use this skill whenever
  the user wants to register an MCP server on ibl.ai, create or rotate an MCP
  server connection (platform/agent/user scoped), wire an MCP server to an
  ibl.ai agent (mentor), set up per-user in-chat OAuth for an MCP tool,
  provision OAuth connected services (Google Drive, Dropbox, etc.), handle
  in-chat MCP events (oauth_required, oauth_connection_resolved, warning), or
  debug MCP auth failures on ibl.ai (401s, "OAuth2 connections require a
  connected service", missing OAuth prompts, agents ignoring MCP servers).
  Trigger even for indirect phrasing like "hook up Google Drive to my ibl
  agent", "add an MCP tool to my mentor", or "why isn't my learner getting
  the OAuth prompt".
---

# ibl.ai MCP Configuration

Configure external tool access for AI agents (mentors) on the ibl.ai platform. The platform models MCP with three objects, and a working integration always requires all three:

1. **MCP Server** — metadata for the external MCP endpoint (name, URL, transport, `auth_type`, `auth_scope`).
2. **MCP Server Connection** — the credential binding (a static token, or a reference to an OAuth `ConnectedService`), at `platform`, `agent`, or `user` scope.
3. **Agent wiring** — the agent must have `"mcp-tool"` in its `tools` list AND the server ID in its `mcp_servers` list.

A missing step 3 is the most common failure mode: server and connection exist but the agent never calls them. Always finish by verifying agent settings.

## Prerequisites (gather before doing anything)

| Value                      | Notes                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------- |
| Base URL                   | e.g. `https://base.manager.iblai.app` (deployment-specific)                              |
| Org / tenant key           | `{org}` in paths, e.g. `acme`                                                            |
| Admin username             | `{user_id}` in paths. **Must be a tenant admin** for all create/update/delete operations |
| API token                  | Sent as `Authorization: Token <value>`                                                   |
| Agent (mentor) `unique_id` | UUID — needed for agent wiring or agent-scoped connections                               |

Never echo tokens back into chat or commit them to files. Read them from environment variables (e.g. `IBL_BASE_URL`, `IBL_ORG`, `IBL_USER`, `IBL_TOKEN`) and interpolate into requests, as the curl examples below do.

## Endpoint map

All agent endpoints live under `/api/ai-agent/orgs/{org}/users/{user_id}/`; OAuth connector endpoints under `/api/accounts/`.

| Capability                       | Endpoint                                                                            | Method              |
| -------------------------------- | ----------------------------------------------------------------------------------- | ------------------- |
| List / create servers            | `/api/ai-agent/orgs/{org}/users/{user_id}/mcp-servers/`                             | GET / POST          |
| Update / delete server           | `/api/ai-agent/orgs/{org}/users/{user_id}/mcp-servers/{id}/`                        | PATCH, PUT / DELETE |
| List / create connections        | `/api/ai-agent/orgs/{org}/users/{user_id}/mcp-server-connections/`                  | GET / POST          |
| Update / delete connection       | `/api/ai-agent/orgs/{org}/users/{user_id}/mcp-server-connections/{id}/`             | PATCH, PUT / DELETE |
| Agent settings (tools + servers) | `/api/ai-agent/orgs/{org}/users/{user_id}/agents/{mentor_id}/settings/`             | GET / PATCH, PUT    |
| List OAuth services              | `/api/accounts/orgs/{org}/oauth-services/`                                          | GET                 |
| Scopes for a service             | `/api/accounts/orgs/{org}/oauth-services/{service_name}/scopes/`                    | GET                 |
| Start OAuth flow                 | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/{provider}/{service}/` | GET                 |
| OAuth callback                   | `/api/accounts/connected-services/callback/`                                        | GET                 |
| List user's connected services   | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/`                      | GET                 |
| Delete connected service         | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/{id}/`                 | DELETE              |

OAuth connector endpoints allow clients only GET/DELETE — `ConnectedService` records are created exclusively by the managed OAuth flow.

## Step 0 — Choose the auth pattern first

The `auth_type` × `auth_scope` decision on the **server** drives everything downstream:

| Pattern                                                     | Server fields                                | Connection setup                                                               | Learner prompted? |
| ----------------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------ | ----------------- |
| No auth                                                     | `auth_type=none`                             | Connection with no credentials                                                 | No                |
| Shared API key for whole tenant                             | `auth_type=token`, `auth_scope=platform`     | One platform-scoped connection with the key                                    | No                |
| Per-agent key                                               | `auth_type=token`, `auth_scope=agent`        | One agent-scoped connection per agent                                          | No                |
| Pre-provisioned per-user                                    | `auth_scope=user`                            | Admin bulk-creates user-scoped connections                                     | No                |
| **In-chat OAuth (each learner connects their own account)** | `auth_type=oauth2` **and** `auth_scope=user` | None upfront — created automatically when the learner completes OAuth mid-chat | **Yes**           |

Facts to keep straight:

- `auth_type` = _how_ credentials go on the wire (none / static token / OAuth2). `auth_scope` = _whose_ credentials are used. Orthogonal.
- `auth_type="oauth2"` + `auth_scope="user"` is the **only** combination that triggers the in-chat OAuth prompt. `oauth2` alone is not enough.
- Any `oauth2` connection (at any scope) **requires** a `connected_service` ID — an existing OAuth grant (see OAuth connector flow below).

## Step 1 — Register the server

`POST /api/ai-agent/orgs/{org}/users/{user_id}/mcp-servers/`

```json
{
  "name": "Google Drive MCP",
  "description": "Search and index Drive documents",
  "url": "https://drive-mcp.example.com",
  "transport": "sse",
  "auth_type": "oauth2",
  "auth_scope": "user",
  "is_featured": false,
  "is_enabled": true
}
```

- `transport`: `sse`, `websocket`, or `streamable_http`.
- `is_featured=true` lets **other tenants** create their own connections to this server (multi-tenant sharing). Original tenant retains control of the metadata.
- `is_enabled=false` is a hard off-switch — disabled servers are skipped at runtime.
- In-chat OAuth servers must also link an `oauth_service` (an `OauthService` record ID).

Capture the returned `id` — the connection and agent wiring both need it.

## Step 2 — Create the connection

`POST /api/ai-agent/orgs/{org}/users/{user_id}/mcp-server-connections/` — payload depends on scope.

**Platform scope (token):**

```json
{
  "server": 9,
  "scope": "platform",
  "auth_type": "token",
  "credentials": "super-secret-api-key",
  "authorization_scheme": "Bearer",
  "extra_headers": { "x-mcp-client": "agent-ui" }
}
```

**Agent scope (token):** add `"agent": "<mentor unique_id UUID>"` and use `"scope": "agent"`. The agent's `platform_key` must match the current tenant; `platform` is inferred from the agent. Use case: different agents present different credentials to the same server (e.g. read/write vs read-only keys).

**User scope (OAuth2):** requires an existing `ConnectedService` (see OAuth flow below).

```json
{
  "server": 9,
  "scope": "user",
  "auth_type": "oauth2",
  "user": "alice",
  "connected_service": 77
}
```

Validation rules the API enforces:

| Scope      | Required                                                               | Forbidden       |
| ---------- | ---------------------------------------------------------------------- | --------------- |
| `platform` | `server`, `auth_type`, credentials **or** `connected_service`          | `user`, `agent` |
| `agent`    | `server`, `auth_type`, `agent`, credentials **or** `connected_service` | `user`          |
| `user`     | `server`, `auth_type`, `user` **or** `connected_service`               | `agent`         |

Plus: `auth_type="oauth2"` always requires `connected_service`, at every scope. Validation errors come back per-field, e.g. `{"connected_service": ["OAuth2 connections require a connected service."]}`.

Token-connection details:

- `authorization_scheme` becomes the header prefix (`Authorization: Bearer <credentials>`); omit it to send the raw value.
- `extra_headers` is an arbitrary JSON object merged into every outbound request; explicit credentials override pre-existing headers.
- `credentials` is **masked on read** (`sup****key`). When PATCHing, only send `credentials` if actually rotating the secret — never send the masked value back.
- Prefer `PATCH {"is_active": false}` over DELETE if the credential may return.
- OAuth-backed connections auto-refresh access tokens near expiry — no client action needed.

Skip this step entirely for the in-chat OAuth pattern — the platform creates the connection itself when the learner authenticates mid-chat.

## Step 3 — Wire the agent

`PATCH /api/ai-agent/orgs/{org}/users/{user_id}/agents/{mentor_id}/settings/`

```json
{ "tools": ["mcp-tool"], "mcp_servers": [9, 14] }
```

**Critical semantics — replace, not merge.** Both fields overwrite the existing value on every update:

- Always `GET` current settings first, then send the **full desired list** including anything already enabled.
- `[]` clears everything. `null` leaves the field untouched.
- Blindly sending `{"tools": ["mcp-tool"]}` silently strips every other tool the agent had.

Safe update procedure (read-merge-write):

```bash
# 1. Read current settings
curl -s -H "Authorization: Token $IBL_TOKEN" \
  "$IBL_BASE_URL/api/ai-agent/orgs/$IBL_ORG/users/$IBL_USER/agents/$MENTOR_ID/settings/" \
  > current.json

# 2. Merge locally: keep existing tools, ensure "mcp-tool" present;
#    keep existing mcp_servers, append the new server ID

# 3. Write back the FULL lists
curl -s -X PATCH \
  -H "Authorization: Token $IBL_TOKEN" -H "Content-Type: application/json" \
  -d '{"tools": ["ai-index", "mcp-tool"], "mcp_servers": [3, 9]}' \
  "$IBL_BASE_URL/api/ai-agent/orgs/$IBL_ORG/users/$IBL_USER/agents/$MENTOR_ID/settings/"
```

## Step 4 — Verify

1. `GET /api/ai-agent/orgs/{org}/users/{user_id}/mcp-servers/` — server present, `is_enabled: true`.
2. `GET /api/ai-agent/orgs/{org}/users/{user_id}/mcp-server-connections/` — connection present, `is_active: true`, correct scope. (Not needed for in-chat OAuth servers.)
3. `GET /api/ai-agent/orgs/{org}/users/{user_id}/agents/{mentor_id}/settings/` — `mcp-tool` in `tools`, server ID in `mcp_servers`.
4. For in-chat OAuth: confirm the admin checklist below is complete.

## Runtime credential resolution

When an agent invokes an MCP server, credentials resolve in this order — first match wins:

1. User-scoped connection for (server, user)
2. Agent-scoped connection for (server, agent)
3. Platform-scoped connection for (server, tenant)
4. Featured-server global fallback
5. Fail 401 / no connection — **or** trigger the in-chat OAuth prompt if the server is `auth_scope="user"` + `auth_type="oauth2"`

Behind the scenes: `MCPServer.resolve_connection(platform, user, agent)` walks this chain, then `render_headers()` refreshes OAuth tokens if needed and merges `extra_headers`. Tenant client credentials come from the credential store via `get_cred("auth_{provider}", tenant)` — tenant overrides live alongside global (`tenant="main"`) entries.

## OAuth connectors (ConnectedService lifecycle)

Terminology: an **OauthProvider** is the vendor (google, dropbox); an **OauthService** is one surface of it (drive, calendar); a **ConnectedService** is a user's persisted token grant for one service — unique on `(user, provider, platform, service)`.

Prerequisite: the tenant must have a credential named `auth_{provider}` (containing `client_id`, `client_secret`, `redirect_uri`) in the credential store before any flow can start. HTTP 400 "No credentials found" on start means this is missing.

Flow:

1. **Discover** — `GET /api/accounts/orgs/{org}/oauth-services/` returns enabled services with `id`, `oauth_provider`, `name`, `display_name`, `scope`, `image`.
2. **Start** — `GET /api/accounts/connected-services/orgs/{org}/users/{user_id}/{provider}/{service}/` returns `{"auth_url": "https://accounts.google.com/o/oauth2/v2/auth?..."}`. Open it in a new tab/popup (providers block iframes). This primes a state cache entry that **expires after 1 hour**.
3. **Callback** — the vendor redirects the browser; relay the query params unmodified to `/api/accounts/connected-services/callback/?code=...&state=...`. Never decode or alter `state` (format: `org:provider:service:user:hash`, verified against the cache). Success returns the `ConnectedService`:

```json
{
  "id": 77,
  "provider": "google",
  "service": "drive",
  "expires_at": "2025-11-12T14:05:00Z",
  "scope_names": ["drive"],
  "token_type": "bearer",
  "service_info": { "id": 12, "name": "drive", "display_name": "Google Drive" }
}
```

If a grant already existed for the same (user, provider, platform, service) it is updated in place. Use `id` as `connected_service` on an MCP connection.

4. **List / delete** — GET/DELETE under `/connected-services/orgs/{org}/users/{user_id}/`. Delete returns `204`.

Token refresh is automatic server-side. `Invalid state` on callback means the round-trip spanned browser contexts or exceeded the 1-hour window — restart the flow. `Could not exchange auth token` means the provider rejected the code — verify the redirect URI matches the provider console.

## In-chat MCP events (per-user OAuth at chat time)

Events arrive as JSON strings on the **existing** chat WebSocket/SSE connection — parse and switch on `type`. Never close or refresh the connection while waiting; resolution arrives on the same socket.

**Trigger conditions** (all must hold): server has `auth_type="oauth2"`, server has `auth_scope="user"`, no valid connection exists for the current user + server, and the chat user is authenticated (non-anonymous).

**Admin setup checklist** (must be complete before any prompt can fire):

1. Create the `OauthProvider` (e.g. `google`) with valid `auth_url`/`token_url`.
2. Create the `OauthService` (e.g. `drive`) linked to the provider with the required `scope`.
3. Store the `auth_{provider}` credential (client_id, client_secret, redirect_uri like `https://your-app.com/api/ai-agent/orgs/main/users/oauth/callback/`) in the credential store.
4. Register the `MCPServer` with `auth_type="oauth2"`, `auth_scope="user"`, `is_enabled=true`, and the linked `oauth_service`. (`auth_scope` can be added later via `PATCH /api/ai-agent/orgs/{org}/users/{user_id}/mcp-servers/{id}/`.)
5. Attach the server to the agent (`tools` + `mcp_servers`).

**Handshake sequence:**

1. Learner sends a message; backend fails to resolve a user connection.
2. Backend emits `oauth_required` (with `auth_url`) and polls the DB every 10s.
3. Frontend opens `auth_url`; user consents; the **backend** callback exchanges the code and creates the `ConnectedService` + `MCPServerConnection` automatically — the frontend does not process the callback.
4. Backend emits `oauth_connection_resolved` and resumes the turn; the normal reply follows.
5. On timeout (default 300s) an `error` (status 400) terminates the turn. On WebSocket transports the connection closes after the error. If the user finishes OAuth _after_ the timeout, their **next message succeeds automatically** — offer a Retry button.

**Event reference:**

| Event `type`                | Key fields                                        | Client action                                                                                                                               |
| --------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `oauth_required`            | `server_name`, `server_id`, `auth_url`, `message` | Show prompt naming the server; open `auth_url` in new tab; show waiting indicator                                                           |
| `oauth_connection_resolved` | `server_name`, `server_id`, `message`             | Dismiss prompt; optional success toast; chat resumes automatically                                                                          |
| `mcp_tools_retrieved`       | `session_id`, `mentor_id`                         | Informational: tool fetch succeeded on retry (3 attempts, backoff 1s/2s/4s). Log or ignore                                                  |
| `warning`                   | `message`, `developer_error`, `code: 503`         | Non-OAuth tool failure; chat continues **without** MCP tools. Show `message` in a banner; log `developer_error`, never show it to end users |
| `error`                     | `error`, `status_code: 400`                       | OAuth timeout / URL build failure / missing connected service. Turn terminates; offer retry                                                 |

**Constants:** `MCP_OAUTH_MAX_WAIT_SECONDS=300`, `MCP_OAUTH_POLL_INTERVAL_SECONDS=10`. Each poll checks for an `MCPServerConnection` with a valid `ConnectedService` for user+server, or a `ConnectedService` matching provider+user+platform — first match resolves.

**Frontend handler pattern:**

```javascript
function handleMessage(data) {
  const message = JSON.parse(data);
  switch (message.type) {
    case "oauth_required":
      showOAuthPrompt({
        serverName: message.server_name,
        serverId: message.server_id,
        authUrl: message.auth_url,
        displayMessage: message.message,
      });
      break;
    case "oauth_connection_resolved":
      dismissOAuthPrompt(message.server_id);
      showToast(`Connected to ${message.server_name}`);
      break;
    case "mcp_tools_retrieved":
      console.debug("MCP tools recovered", message);
      break;
    case "warning":
      showWarningBanner(message.message);
      console.warn("MCP warning:", message.developer_error);
      break;
    default:
      if (message.error && message.status_code) handleError(message);
  }
}
```

## API quick reference (curl)

```bash
export IBL_BASE_URL=https://base.manager.iblai.app
export IBL_ORG=acme IBL_USER=alice IBL_TOKEN=xxxx
AUTH="Authorization: Token $IBL_TOKEN"
JSON="Content-Type: application/json"
AGENT_API="$IBL_BASE_URL/api/ai-agent/orgs/$IBL_ORG/users/$IBL_USER"
ACCOUNTS_API="$IBL_BASE_URL/api/accounts"

# --- MCP servers ---
curl -s -H "$AUTH" "$AGENT_API/mcp-servers/"                       # list
curl -s -X POST -H "$AUTH" -H "$JSON" "$AGENT_API/mcp-servers/" \  # create
  -d '{"name":"Workflow MCP","url":"https://wf.example.com","transport":"sse",
       "auth_type":"token","auth_scope":"platform","is_enabled":true}'
curl -s -X PATCH -H "$AUTH" -H "$JSON" \                           # update (e.g. enable in-chat OAuth)
  "$AGENT_API/mcp-servers/9/" -d '{"auth_scope":"user","auth_type":"oauth2"}'
curl -s -X DELETE -H "$AUTH" "$AGENT_API/mcp-servers/9/"           # delete

# --- MCP connections ---
curl -s -H "$AUTH" "$AGENT_API/mcp-server-connections/"            # list
curl -s -X POST -H "$AUTH" -H "$JSON" \                            # create (platform token)
  "$AGENT_API/mcp-server-connections/" \
  -d "{\"server\":9,\"scope\":\"platform\",\"auth_type\":\"token\",
       \"credentials\":\"$MCP_KEY\",\"authorization_scheme\":\"Bearer\"}"
curl -s -X POST -H "$AUTH" -H "$JSON" \                            # create (user OAuth2)
  "$AGENT_API/mcp-server-connections/" \
  -d '{"server":9,"scope":"user","auth_type":"oauth2","user":"alice","connected_service":77}'
curl -s -X PATCH -H "$AUTH" -H "$JSON" \                           # deactivate (prefer over delete)
  "$AGENT_API/mcp-server-connections/12/" -d '{"is_active":false}'
curl -s -X DELETE -H "$AUTH" "$AGENT_API/mcp-server-connections/12/"

# --- Agent wiring (read first — tools/mcp_servers are REPLACED, not merged) ---
curl -s -H "$AUTH" "$AGENT_API/agents/$MENTOR_ID/settings/"
curl -s -X PATCH -H "$AUTH" -H "$JSON" \
  "$AGENT_API/agents/$MENTOR_ID/settings/" \
  -d '{"tools":["mcp-tool"],"mcp_servers":[9]}'

# --- OAuth connectors ---
curl -s -H "$AUTH" "$ACCOUNTS_API/orgs/$IBL_ORG/oauth-services/"                    # discover services
curl -s -H "$AUTH" "$ACCOUNTS_API/orgs/$IBL_ORG/oauth-services/drive/scopes/"       # scopes for a service
curl -s -H "$AUTH" \                                                                # start flow -> {"auth_url": ...}
  "$ACCOUNTS_API/connected-services/orgs/$IBL_ORG/users/$IBL_USER/google/drive/"
curl -s -H "$AUTH" "$ACCOUNTS_API/connected-services/orgs/$IBL_ORG/users/$IBL_USER/"      # list grants
curl -s -X DELETE -H "$AUTH" "$ACCOUNTS_API/connected-services/orgs/$IBL_ORG/users/$IBL_USER/77/"  # revoke (204)
```

The OAuth callback (`GET $ACCOUNTS_API/connected-services/callback/?code=...&state=...`) is hit by the user's browser after provider consent — relay the vendor's query params unmodified; do not call it directly with fabricated values.

## Troubleshooting

| Symptom                                                           | Fix                                                                                                                                                        |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `400 OAuth2 connections require a connected service.`             | Complete the OAuth connector flow first; pass the resulting `connected_service` ID.                                                                        |
| `400 Selected MCP server is not available to the current tenant.` | Use a server on this tenant, or mark the source server `is_featured=true`.                                                                                 |
| `400 No credentials found`                                        | Tenant admin must install the `auth_{provider}` credential (client_id, client_secret, redirect_uri).                                                       |
| Agent never calls the server                                      | `mcp-tool` missing from `tools`, or server ID missing from `mcp_servers`. These fields are replaced, not merged — a careless PATCH may have stripped them. |
| No OAuth prompt for a per-user server                             | Server needs **both** `auth_scope="user"` and `auth_type="oauth2"`, plus a linked `oauth_service`, plus an authenticated (non-anonymous) chat session.     |
| Prompt fires every message even after auth                        | The `ConnectedService` belongs to a different user or tenant than the chat user.                                                                           |
| Connection unexpectedly falls back to platform creds              | Check the user connection's `is_active` and that `ConnectedService.user` matches the chat user.                                                            |
| `/oauth-services/` returns `[]`                                   | No enabled `OauthService` records / provider disabled. Seed `OauthProvider` + `OauthService`.                                                              |
| Callback `Invalid state`                                          | Start/callback in different browser contexts, or state expired (>1h). Redo the flow in one session.                                                        |
| Callback `Could not exchange auth token`                          | Provider rejected the code — verify the redirect URI matches the provider console; restart.                                                                |
| OAuth timeout in chat                                             | Default wait is 300s. If the user finishes OAuth after timeout, their next message succeeds automatically.                                                 |
| Tool call fails silently                                          | A `warning` (503) event was ignored — surface it; verify the MCP server is reachable from the platform.                                                    |
