# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

`iblai/api` is a toolkit for operating the ibl.ai platform from an AI agent. It ships two things:

1. **Skills** (`skills/`) — installable via `npx skills add iblai/api`. Each `/iblai-*` skill maps one agent-configuration or platform-admin operation to its exact `api.iblai.app` REST endpoints (method, URL, body). They drive the platform directly — no UI.
2. **MCP server** (`mcp/`) — a single hosted Model Context Protocol server, `iblai-agent-chat`, for the one **runtime** capability that isn't a REST call: holding a live conversation with a deployed agent (streamed responses, tool use, RAG).

This is the API-driven complement to [`iblai/vibe`](https://github.com/iblai/vibe), which provides UI components for *building* ibl.ai apps.

**Naming.** Skills are `iblai-*`; scope is encoded by prefix. **`iblai-agent-*`** acts on one agent; **`iblai-profile*`** on the signed-in user. **Org-wide is the default scope, so those skills are bare** — `iblai-management`, `iblai-rbac`, `iblai-crm`, `iblai-tokens`, `iblai-integrations`, `iblai-notifications`, `iblai-invites` — and `iblai-org` itself is the org-settings skill. Cross-cutting skills are bare too: `iblai-search`, `iblai-analytics`, `iblai-course-create`, `iblai-login`. (These `iblai-*` names overlap some of vibe's; that's accepted.)

**Skills describe APIs, not UIs.** Each skill documents endpoints (method, URL, body) and when to use them. Do **not** reference user interfaces (menus, tabs, buttons, pages, "Edit Agent → …") in skill prose — the UI was only a means to discover endpoints; the value is the API.

**Skills vs MCP server — REST vs runtime chat.** Skills do everything reachable over REST (configure agents, datasets, memory, users, roles, notifications, discovery/search, recommendations, user profile + analytics, reporting). The MCP server is only for live agent chat — and even *wiring it up* is a skill (`/iblai-agent-chat` writes the `.mcp.json` / `claude mcp add` config from the `.env` token + a chosen agent's `unique_id`), so the whole offering installs via `npx skills add iblai/api`. **Rule: if a skill covers it, there is no server for it.** That's why only `iblai-agent-chat` remains — the former `analytics`, `agent-create`, `search`, and `user` servers were removed once skills covered analytics (`/iblai-analytics`), agent creation (`/iblai-agent-create`), discovery + recommendations (`/iblai-search`), and user profile + analytics (`/iblai-profile`).

## Terminology (settled — use consistently)

Three words used to be mixed for a customer's workspace. The convention for this repo:

- **Platform** — the ibl.ai system as a whole (`api.iblai.app`, `login.iblai.app`). One platform serves every customer. **Never** use "platform" to mean a single customer's workspace. (It is fine in product terms like "Platform API Token" and in prose like "the platform API".)
- **Organization (org)** — one customer's isolated workspace (its own users, agents, branding, data). This is the **primary noun** in all prose and docs. It matches what customers see on `login.iblai.app/me` ("Organizations") and the API path `/orgs/{org}/`.
- **org key** — the organization's identifier, e.g. `enterprise`. On the API wire it also appears as `org`, `platform_key`, and `platform_org` — **keep those verbatim** in endpoint references; they all mean *the org key*.
- **tenant** — architectural adjective only ("multi-tenant", "tenant isolation"). Do **not** use it as the workspace noun. The only literal `tenant` allowed is the API scope enum value `user|mentor|tenant` (which equals org-wide). Note the `@iblai` SDK / vibe / os use `tenant` in env vars (`NEXT_PUBLIC_TENANT`); this repo deliberately uses `org` instead.

Env vars follow the noun: active workspace is `IBLAI_ORG` + `IBLAI_API_KEY`; saved per-workspace keys are `IBLAI_ORG_<NAME>_KEY`.

## Layout

```
skills/                 # downloadable skills (npx skills add iblai/api)
  iblai-login/      # connect an organization — run first
  iblai-agent-*/    # one-agent skills (settings, llm, prompts, datasets, memory, evals, …)
  iblai-profile*/   # signed-in user's profile + per-user metadata
  iblai-<other>/    # org-wide ops (management, rbac, crm, tokens, …) + search, analytics, course-create
mcp/                    # hosted Python MCP server — runtime chat only
  iblai-agent-chat/
.env                    # active organization + per-org Api-Tokens (gitignored)
```

## Skills

- Each skill is a single `SKILL.md` with YAML frontmatter (`name`, `description`) following the format of `skills/iblai-agent-settings/SKILL.md` and `skills/iblai-login/SKILL.md`.
- **Canonical section structure (every endpoint-documenting skill MUST follow this):**
  1. `## Auth & conventions` — base URL, header, path vars, prefix, "run `/iblai-login` first" line, and the destructive-confirm note.
  2. *(optional)* one short explanatory section (e.g. `## Concepts`, `## Pagination`) when the API needs framing before the endpoints.
  3. `## Reads` — every read endpoint (**GET**/**HEAD**).
  4. `## Writes` — every write endpoint (**POST**/**PUT**/**PATCH**/**DELETE**); mark each destructive/outward-facing call "Confirm with the user first."
  5. `## Example` — one realistic `curl`.
  6. `## Notes` — gotchas.
  - **Multi-resource skills** (catalog, crm, rbac, billing, …) keep their resource grouping as `###` sub-headings **inside** `## Reads` and `## Writes` (a resource with both appears under each). Do **not** group endpoints by resource at the top level — Reads/Writes is always the top-level split.
  - A read-only skill may omit `## Writes`; a write-only skill may omit `## Reads`.
  - Exceptions: non-REST flow skills (`iblai-login`, `iblai-agent-chat`) are setup guides, not endpoint references, and do not use Reads/Writes.
- **Auth model (every skill):** base URL `https://api.iblai.app`, header `Authorization: Api-Token $IBLAI_API_KEY`. Path vars `{org}` = `$IBLAI_ORG` (a.k.a. `platform_key`), `{username}` = `$IBLAI_USERNAME`, `{mentor}` = the agent's unique id.
- **Gateway prefixes — `/dm` and `/edx`.** `api.iblai.app` is a gateway that sits **in front of** the backend services and strips a prefix before routing:
  - `https://api.iblai.app/dm/...` → the **Data Manager** service (the [`iblai/iblai-dm-pro`](https://github.com/iblai/iblai-dm-pro) repo).
  - `https://api.iblai.app/edx/...` → the **Open edX** service (a different system).

  The prefix is added at the gateway, so it does **not** appear in any backend `urls.py` — the source repos register bare `/api/...` routes. A skill must prepend the right prefix (e.g. a DM route `/api/catalog/courses/` is documented and called as `https://api.iblai.app/dm/api/catalog/courses/`). When in doubt, an endpoint is a **`/dm`** endpoint.
- **Source of truth = the repo URLconf, not the docs.** Skills for DM features are derived from `iblai-dm-pro`. The app `USAGE.md` files are a starting point but contain errors, gaps, and (critically) endpoints that only apply to **edX** — those do **not** belong here. **Rule: only document an endpoint if it is registered in that repo's `urls.py` (i.e. reachable at `api.iblai.app/dm/...`).** If a path is not in this repo's URL configuration, drop it — it would not resolve via `/dm`. Always verify each endpoint's method, path, and request fields against the actual `urls.py` / views / serializers before shipping a skill.
- **Connecting an organization:** `/iblai-login` opens `https://login.iblai.app/me`, lets the user pick one of their organizations, and writes `IBLAI_ORG`, `IBLAI_USERNAME`, and `IBLAI_API_KEY` to `.env`. Always ask the user which org to target — accounts can belong to many (40+ is normal).
  - **Logged out:** `/me` redirects to `/login`. Detect this (URL is `/login`, no "My Account" content), hand the user the `https://login.iblai.app/me` URL, and wait for them to sign in — never enter their credentials.
  - **After login the platform redirects somewhere else** (the destination varies and may change), NOT back to `/me`. Don't depend on where it lands — always **re-navigate explicitly to `https://login.iblai.app/me`** before reading params.
  - `/me` is **server-rendered** — there is no JSON API; read org keys + username from the rendered page content. Each org block is the display name followed by its key (e.g. `Enterprise → enterprise`, `ibl.ai → iblai`, or a UUID).
  - There is **no working logout route** (`/logout`, `/api/auth/logout` both 4xx); the session is an httpOnly cookie. To force a logged-out state for testing, clear the browser's cookies for `login.iblai.app`.
- Mark every DELETE / destructive / outward-facing call (delete, send, invite) "confirm with the user first."

## MCP servers

```bash
# Install dependencies
cd mcp/iblai-<service>
uv sync

# Run the server
uv run iblai-<service>

# Run tests
uv run pytest
```

Each server contains:

- **server.py**: tool definitions via `@server.tool()` decorators
- **auth.py**: `AuthManager` supporting api_key, bearer, basic, custom_header, oauth2_client_credentials via env vars
- **client.py**: `APIClient` for async httpx requests

Conventions: server names `iblai-<service>`, package names `iblai_<service>`; stdio transport locally, streamable-http for hosted. The hosted servers expect `Authorization: Api-Token <key>`; the `org` is derived from the token.

## Creating New MCP Servers

Use the [iblai-mcp-creator](https://github.com/iblai/iblai-mcp-creator) tool to generate new MCP servers from HAR files.
