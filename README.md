<div align="center">

<a href="https://ibl.ai"><img src="https://ibl.ai/images/iblai-logo.png" alt="ibl.ai" width="300"></a>

# iblai/api

Operate any ibl.ai organization from your AI agent. Skills + a chat MCP server.

[![Skills](https://img.shields.io/badge/Skills-33-CC785C)](https://skills.sh/iblai/api)
[![Claude Code](https://img.shields.io/badge/Claude_Code-CC785C?logoColor=white)](https://claude.ai)
[![Cursor](https://img.shields.io/badge/Cursor-000000)](https://cursor.com)
[![GitHub Copilot](https://img.shields.io/badge/GitHub_Copilot-000000?logo=githubcopilot&logoColor=white)](https://github.com/features/copilot)
[![MCP](https://img.shields.io/badge/MCP-Model_Context_Protocol-1f6feb)](https://modelcontextprotocol.io)
[![ibl.ai](https://img.shields.io/badge/ibl.ai-platform-000000)](https://ibl.ai)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

> **Note:** These skills and servers run against the hosted `api.iblai.app` environment. If you'd like a license to the full platform codebase to run locally or self-host, reach out to our team at [ibl.ai/contact](https://ibl.ai/contact).

---

## Quick Start

**Before you start:** you need [Node.js](https://nodejs.org) (for `npx`), a skills-compatible AI agent ([Claude Code](https://claude.ai/code), Cursor, OpenCode, and [15+ others](https://skills.sh)), and an ibl.ai account. **No account yet?** Sign up at [ibl.ai/join](https://ibl.ai/join) — it walks you through creating your own organization.

### 1. Install the skills

```bash
npx skills add iblai/api
```

This installs skills that teach your AI agent how to **drive the ibl.ai platform REST API directly** — configure agents, manage datasets and memory, administer users and roles, send notifications, and pull analytics — for any organization you belong to. You'll now have the `/iblai-*` commands in your agent.

### 2. Connect an organization

Run the login skill once. It opens [login.iblai.app/me](https://login.iblai.app/me), you sign in via SSO, it asks which of your organizations to use, and it writes your **org key**, **username**, and a **Platform API Token** to `.env`:

```text
/iblai-login
```

> **First time?** No ibl.ai account → start at [ibl.ai/join](https://ibl.ai/join) (it leads you through creating your organization), then run `/iblai-login`. Already signed in but only see the shared **`main`** organization? That's not your own workspace — create your own org first; `/iblai-login` handles it.

Every other skill then reads `IBLAI_ORG`, `IBLAI_USERNAME`, and `IBLAI_API_KEY` from `.env` and calls `https://api.iblai.app` with `Authorization: Api-Token <key>`.

## What is iblai/api

A toolkit for operating the [ibl.ai](https://ibl.ai) platform from your AI agent. Where [`iblai/vibe`](https://github.com/iblai/vibe) gives you UI components to *build* an app, `iblai/api` gives you skills to *run the platform itself* — every agent-configuration and platform-admin operation mapped to its exact REST endpoints (method, URL, body) — plus a hosted MCP server for the one runtime capability that isn't a REST call: chatting with a deployed agent.

**Why it matters:**

- **One skill per operation** — each `/iblai-*` skill maps one capability, so changing an agent's LLM or pulling cost analytics is a `/` command, not a docs hunt
- **Any organization** — authenticate once via `login.iblai.app/me`, then target any of your organizations by org key + Api-Token
- **Endpoint-accurate** — skills carry the real `api.iblai.app` request shapes, so your agent calls the platform correctly the first time
- **Runtime chat included** — a hosted Model Context Protocol server for chatting with deployed agents (streamed responses, tool use, RAG) — no local install
- **No UI required** — drive the platform headless, from CI, a terminal, or any MCP-capable assistant

## How It Works

1. **Install** — `npx skills add iblai/api` drops the skills into your project.
2. **Connect** — run `/iblai-login` to capture your org + username from `login.iblai.app/me` and store an Api-Token in `.env`.
3. **Operate** — invoke any `/iblai-*` skill; it fills in your org, username, and (where relevant) the agent id, then calls `api.iblai.app`.
4. **Automate** — chain skills, or connect the hosted chat MCP server to your assistant to talk to agents at runtime.

## Skills

After installing, use these directly in your AI agent with `/` commands.

### Setup

```text
/iblai-login
```

### Agent

```text
/iblai-agent-create        /iblai-agent-datasets
/iblai-agent-settings      /iblai-agent-embed
/iblai-agent-sandbox       /iblai-agent-memory
/iblai-agent-access        /iblai-agent-history
/iblai-agent-llm           /iblai-agent-audit
/iblai-agent-prompts       /iblai-agent-evals
/iblai-agent-skills        /iblai-agent-chat
/iblai-agent-safety        /iblai-agent-disclaimers
/iblai-agent-privacy       /iblai-agent-tools
/iblai-agent-mcp
```

### Organization (platform admin)

```text
/iblai-org                 /iblai-rbac
/iblai-management       /iblai-crm
/iblai-integrations     /iblai-notifications
/iblai-tokens          /iblai-invites
/iblai-scim            /iblai-billing
/iblai-features
```

### Profile

```text
/iblai-profile             /iblai-profile-metadata
```

### Content & discovery

```text
/iblai-search              /iblai-course-create
/iblai-analytics           /iblai-catalog
/iblai-milestones          /iblai-credentials
/iblai-catalog-media       /iblai-catalog-invitations
```

### What each skill does

| Skill | Description |
|-------|-------------|
| `/iblai-login` | Connect an organization — opens `login.iblai.app/me`, captures org + username + Api-Token into `.env`. Run first. |
| `/iblai-agent-create` | Create a new agent from a template (then configure with the other agent skills) |
| `/iblai-agent-settings` | Agent identity & capabilities — name, description, category, image, visibility, flags; fork and delete |
| `/iblai-agent-sandbox` | Connect/disconnect a sandbox (Claw) instance, push config, run health checks, set the model |
| `/iblai-agent-access` | Role-based access to an agent — grant editor / chat / analytics_viewer to users, groups, emails |
| `/iblai-agent-llm` | Choose the agent's LLM provider and model |
| `/iblai-agent-prompts` | System / proactive / study / guided prompts and suggested-prompt CRUD |
| `/iblai-agent-skills` | Browse the skill catalog and assign skills to an agent (requires a sandbox) |
| `/iblai-agent-safety` | Moderation & safety systems, prompts/responses, and flagged-prompt logs |
| `/iblai-agent-privacy` | Privacy Router — PII detection, redact/mask/block, entity types, output filtering |
| `/iblai-agent-disclaimers` | Advisory text and the User Agreement |
| `/iblai-agent-tools` | Enable/disable the agent's tools |
| `/iblai-agent-mcp` | MCP connectors — create, edit, enable, and complete OAuth connections |
| `/iblai-agent-datasets` | Training datasets (RAG) — add files/URLs/YouTube/crawl/GitHub, train, retrain, delete |
| `/iblai-agent-embed` | Embed/widget settings + backend token provisioning — CSS/JS, voice, SSO, share links |
| `/iblai-agent-memory` | Agent memories and memory categories — list, filter, add, edit, delete |
| `/iblai-agent-history` | Conversation history & summaries; export chat history as an async report |
| `/iblai-agent-audit` | Agent audit log — who changed what (read-only) |
| `/iblai-agent-evals` | Agent evaluations — datasets, experiments, LLM-as-Judge + human scoring, CSV export |
| `/iblai-agent-chat` | Set up live chat with an agent — wires the `iblai-agent-chat` MCP server into the project |
| `/iblai-org` | Org-wide settings — default agent, help center URL, chat width, feature toggles |
| `/iblai-management` | Org admin — Users, Groups, Roles, Policies, Teams, Alerts |
| `/iblai-rbac` | RBAC — roles, policies, groups, permission checks, agent/team sharing, student toggles |
| `/iblai-crm` | CRM — people, organizations, pipelines, deals (move/won/lost), activities, tags |
| `/iblai-integrations` | Account integrations — LLM keys, Data Source credentials, API tokens |
| `/iblai-tokens` | Platform API Tokens — list, create (secret shown once), delete |
| `/iblai-notifications` | Org notifications — counts, inbox, mark-as-read, build & send |
| `/iblai-invites` | User invitations — list and send (single or CSV bulk) |
| `/iblai-scim` | SCIM 2.0 directory provisioning — users (Enterprise extension), groups, departments, memberships; RBAC group assignment auto-links platforms |
| `/iblai-billing` | Billing & credits — credit accounts, item paywalls, prices, checkout (auth + guest), subscriptions, access checks, revenue/subscriber reporting |
| `/iblai-features` | Per-user feature config & flags — get/update (inline or feature+values), bulk-config, apps/onboarding, trial activation, platform provisioning |
| `/iblai-profile` | The signed-in user's own profile — Basic, Social, Education, Experience, Resume, Memory |
| `/iblai-profile-metadata` | Per-user, per-org metadata key-value store — preferences, settings, feature flags |
| `/iblai-search` | Discover agents and content + personalized (RAG) recommendations — faceted search (read-only) |
| `/iblai-analytics` | Analytics across agents, content, and users — KPIs, users, topics, transcripts, costs, courses, programs, audit, reports |
| `/iblai-course-create` | Course Creation API — generate, edit, and publish courses (tasks, outline, structure) |
| `/iblai-catalog` | Learning catalog — courses, programs, pathways, resources, skills/roles taxonomy, enrollment, eligibility, reviews |
| `/iblai-milestones` | Catalog milestones — course/resource/program/pathway completions and skill points (block, course, platform, user) |
| `/iblai-credentials` | Digital credentials — credential CRUD, user/group assignments, assertions, course import/export, provider config (Accredible), analytics |
| `/iblai-catalog-media` | Catalog media resources — list/create/update/delete media tied to courses/units/items, multipart upload, search, by-item lookup |
| `/iblai-catalog-invitations` | Catalog invitations & licensing — platform/course/program invitations (bulk, blank, redeem), licenses & assignments, access requests, suggestions |

Skills live in [`skills/`](./skills). Read them, extend them, or write your own.

## MCP Server

Hosted Model Context Protocol server — no local installation required. This covers the one **runtime** capability the skills can't: actually talking to a deployed agent. Wire it up with **`/iblai-agent-chat`** (it writes the config below from your `.env` token + a chosen agent), or add it manually. (Administering the platform is the skills' job — see [Skills vs MCP servers](#skills-vs-mcp-servers).)

| Server | Description | Endpoint |
|--------|-------------|----------|
| [iblai-agent-chat](./mcp/iblai-agent-chat) | Talk to a deployed agent — streamed AI responses (runtime) | `/mcp/agent-chat/` |

### Connect from Claude Code

```bash
claude mcp add iblai-agent-chat --transport http https://asgi.data.iblai.app/mcp/agent-chat/ --header "Authorization: Api-Token YOUR_API_TOKEN"
```

### Connect from Claude Desktop / Cursor

```json
{
  "mcpServers": {
    "iblai-agent-chat": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN"
      }
    }
  }
}
```

### Skills vs MCP servers

The split is **administer/use-via-REST vs chat at runtime**:

- **Skills** do everything you can reach over REST — create and configure agents, manage datasets, memory, users, roles, notifications, discovery/search, recommendations, user profiles and analytics, and reporting — by calling the API directly.
- **The MCP server** is for the one thing that isn't a REST admin call: holding a live conversation with a deployed agent (streamed responses, tool use, RAG).

**Rule: if a skill covers it, there is no server for it.** That's why only `iblai-agent-chat` remains — the former `analytics`, `agent-create`, `search`, and `user` servers were removed once skills covered analytics, agent creation, discovery + recommendations (`/iblai-search`), and user profile + analytics (`/iblai-profile`).

## Authentication

Everything authenticates the same way:

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token <key>` on every request
- **Org & username:** from [login.iblai.app/me](https://login.iblai.app/me) — each organization you belong to shows its **key** (e.g. `enterprise`, `iblai`, or a UUID)
- **Api-Token:** your **first** token comes from the platform admin (or `login.iblai.app`); once you're connected, `/iblai-tokens` lists, creates, and rotates tokens. The secret is shown once.

The `/iblai-login` skill walks through all of this and writes `IBLAI_ORG`, `IBLAI_USERNAME`, and `IBLAI_API_KEY` to `.env`. Never commit `.env` — it is in `.gitignore`.

## Resources

- [skills.sh/iblai/api](https://skills.sh/iblai/api) — install skills with `npx skills add iblai/api`
- [iblai/vibe](https://github.com/iblai/vibe) — companion toolkit for *building* ibl.ai apps (UI components + Claude Code skills)
- [login.iblai.app/me](https://login.iblai.app/me) — your account, organizations, and org keys
- [docs.ibl.ai](https://docs.ibl.ai) — platform documentation

## License

MIT — see [LICENSE](LICENSE). © [ibl.ai](https://ibl.ai)
