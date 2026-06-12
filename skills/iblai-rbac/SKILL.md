---
name: iblai-rbac
description: Manage an ibl.ai organization's RBAC via the platform API — roles, policies, groups, permission checks, agent and team access sharing, bulk user policies, and student creation/LLM-access toggles. Org-wide access control. Use to define who can do what.
---

# iblai-rbac

Drive the organization's **role-based access control** from the API: define
roles and policies, attach them to groups and users, check permissions, share
agents and teams, and toggle what students may do — the org-wide "who can do
what" surface under `…/dm/api/core/rbac/…`.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG` (a.k.a. `platform_key`),
  `{username}` = `$IBLAI_USERNAME`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.
- The RBAC developer docs phrase auth as `Authorization: Token <key>` — that is
  the same platform key; use `Api-Token`.

## Concepts

- **Resource paths are hierarchical** and prefixed with `/platforms/{pk}/` — a
  policy granted on a parent resource applies to its children.
- **Actions** look like `Ibl.Agent/Agents/delete` (namespace / resource / verb).
- There are **well-known roles** the platform ships with; list them with the
  roles endpoint (optionally including globals).

## Roles

- **GET** `https://api.iblai.app/dm/api/core/rbac/roles/?platform_key={org}` — list roles (add `&include_global_roles=true` to include globals).
- **POST** `https://api.iblai.app/dm/api/core/rbac/roles/` — create a role.
- **GET** `https://api.iblai.app/dm/api/core/rbac/roles/{id}/?platform_key={org}` — fetch one role.
- **PUT / PATCH** `https://api.iblai.app/dm/api/core/rbac/roles/{id}/` — update a role.
- **DELETE** `https://api.iblai.app/dm/api/core/rbac/roles/{id}/?platform_key={org}` — delete a role. Destructive — confirm with the user first.

## Policies

- **GET / POST** `https://api.iblai.app/dm/api/core/rbac/policies/?platform_key={org}` — list / create policies.
- **GET** `https://api.iblai.app/dm/api/core/rbac/policies/{id}/?platform_key={org}` — fetch one policy.
- **PUT / PATCH** `https://api.iblai.app/dm/api/core/rbac/policies/{id}/` — update a policy.
- **DELETE** `https://api.iblai.app/dm/api/core/rbac/policies/{id}/?platform_key={org}` — delete a policy. Destructive — confirm with the user first.

## Groups

- **GET / POST** `https://api.iblai.app/dm/api/core/rbac/groups/?platform_key={org}` — list / create groups.
- **GET** `https://api.iblai.app/dm/api/core/rbac/groups/{id}/?platform_key={org}` — fetch one group.
- **PUT / PATCH** `https://api.iblai.app/dm/api/core/rbac/groups/{id}/` — update a group.
- **DELETE** `https://api.iblai.app/dm/api/core/rbac/groups/{id}/?platform_key={org}` — delete a group. Destructive — confirm with the user first.

## Permission check

- **POST** `https://api.iblai.app/dm/api/core/rbac/permissions/check/` — check the caller's access to resources:
  ```json
  {
    "platform_key": "string (required)",
    "resources": ["/users/", "/groups/"]
  }
  ```

## Agent access

- **POST** `https://api.iblai.app/dm/api/core/rbac/agent-access/` — share an agent (body includes `platform_key`, `mentor_id`, `role`, and `users`/`groups`).
- **GET** `https://api.iblai.app/dm/api/core/rbac/agent-access/?platform_key={org}&mentor_id={id}` — list an agent's access policies.

## Team (user-group) sharing

- **POST** `https://api.iblai.app/dm/api/core/rbac/teams/access/` — share a user-group (team).
- **GET** `https://api.iblai.app/dm/api/core/rbac/teams/access/?platform_key={org}&usergroup_id={id}` — list a team's access policies.

## Bulk user policies

- **PUT** `https://api.iblai.app/dm/api/core/platform/users/policies/` — set policies for users (array).

## Student toggles

- **POST** `https://api.iblai.app/dm/api/core/rbac/student-agent-creation/set/` — enable/disable student agent creation.
- **GET** `https://api.iblai.app/dm/api/core/rbac/student-agent-creation/status/?platform_key={org}` — read the current toggle.
- **POST** `https://api.iblai.app/dm/api/core/rbac/student-llm-access/set/` — enable/disable student LLM access.
- **GET** `https://api.iblai.app/dm/api/core/rbac/student-llm-access/status/?platform_key={org}` — read the current toggle.

## Example

Check whether the caller can act on users and groups in the org:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/core/rbac/permissions/check/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"platform_key\": \"$IBLAI_ORG\", \"resources\": [\"/users/\", \"/groups/\"]}"
```

## Notes

- This is the **authoritative RBAC surface** for the organization. The
  `/iblai-management` and `/iblai-agent-access` skills touch the same
  roles / policies / agent-access endpoints from their admin views — cross-reference
  them when working across surfaces.
- Resource paths are hierarchical (`/platforms/{pk}/…`): a policy on a parent
  grants its children, so scope policies at the right level.
- Most reads carry `platform_key={org}` as a query param; writes (POST/PUT/PATCH)
  carry `platform_key` in the body.
