---
name: iblai-management
description: Administer an ibl.ai organization via the platform API — manage Users (list, activate, promote/demote, set policies), Groups, RBAC Roles and Policies, Teams (user-groups + team access), and Alerts (watched groups/users/watchers). Use for organization-level user and access administration.
---

# iblai-management

Administer an organization from the API: manage **Users, Groups, Roles,
Policies, Teams, and Alerts** for organization-level user and access
administration.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG` (a.k.a. `platform_key`),
  `{username}` = `$IBLAI_USERNAME`.
- **Host:** these endpoints live on the DM host under `/api/core/…`. The one
  exception is user promote/demote, which flips to the LMS host
  `learn.iblai.app`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads & Writes

### Users

- **GET** `…/core/platform/users/?platform_key={org}&platform_org={org}&query={q}&page={n}&page_size=10&return_policies=true` — user list + policies.
- **POST** `https://learn.iblai.app/api/ibl/users/manage/roles/` — promote/demote (LMS):
  ```json
  {
    "username": "string (required)",
    "org": "string (required)",
    "role": "e.g. org-instructor (required)",
    "active": "boolean (required)"
  }
  ```
- **POST** `…/core/users/platforms/` — activate/deactivate:
  ```json
  {
    "user_id": "number (required)",
    "platform_key": "string (required)",
    "active": "boolean (required)"
  }
  ```
- **PUT** `…/core/platform/users/policies/` — set policies (array):
  ```json
  [
    {
      "user_id": "number (required)",
      "platform_key": "string (required)",
      "policies_to_set": "string[] (required, may be [])"
    }
  ]
  ```

### Groups

`…/core/rbac/groups/`

- **GET** `…/groups/?platform_key={org}&include_users=true[&name=&email=&owner=&username=&page=&page_size=]` — list.
- **GET** `…/groups/{id}/` — detail.
- **POST** `…/groups/` — create:
  ```json
  {
    "name": "string (required)",
    "platform_key": "string (required)",
    "description": "string",
    "users": "number[]"
  }
  ```
- **PUT** `…/groups/{id}/` — update (same shape).
- **DELETE** `…/groups/{id}/?platform_key={org}` — delete. Destructive — confirm with the user first.

### Roles

`…/core/rbac/roles/` (all calls include `include_global_roles=true`)

- **GET** `…/roles/?include_global_roles=true&platform_key={org}[&name=&page=&page_size=]` — list.
- **GET** `…/roles/{id}/?include_global_roles=true&platform_key={org}` — detail.
- **POST** `…/roles/?include_global_roles=true` — create:
  ```json
  {
    "name": "string (required)",
    "platform_key": "string (required)",
    "description": "string",
    "permissions": "string[]"
  }
  ```
- **PUT** / **PATCH** `…/roles/{id}/?include_global_roles=true` — update (same shape).
- **DELETE** `…/roles/{id}/?include_global_roles=true&platform_key={org}` — delete. Destructive — confirm with the user first.

### Policies

`…/core/rbac/policies/`

- **GET** `…/policies/?platform_key={org}&include_groups=true&include_users=true[&role_id=&group=&name=&username=&email=&page=&page_size=]` — list.
- **GET** `…/policies/{id}/?platform_key={org}` — detail.
- **POST** `…/policies/` — create:
  ```json
  {
    "name": "string (required)",
    "platform_key": "string (required)",
    "role": "number role id (required)",
    "resources": "string[]",
    "users": "number[]",
    "groups": "number[]"
  }
  ```
- **PUT** / **PATCH** `…/policies/{id}/` — update (same shape).
- **DELETE** `…/policies/{id}/?platform_key={org}` — delete. Destructive — confirm with the user first.

### Teams

`…/core/user-groups/` plus team access (`…/core/rbac/teams/access/`)

- **GET** `…/user-groups/?platform_key={org}[&include_users=&name=&with_permissions=&page=&page_size=]` — list.
- **GET** `…/user-groups/{id}/?platform_key={org}` — detail.
- **POST** `…/user-groups/` — create:
  ```json
  {
    "name": "string (required)",
    "platform_key": "string (required)",
    "description": "string",
    "users": "number[]"
  }
  ```
- **PUT** `…/user-groups/{id}/` — update (same shape).
- **DELETE** `…/user-groups/{id}/` — delete. Destructive — confirm with the user first.
- **GET** `…/core/rbac/teams/access/?platform_key={org}&usergroup_id={id}` — list a team's access policies.
- **POST** `…/core/rbac/teams/access/` — set team access:
  ```json
  {
    "platform_key": "string (required)",
    "usergroup_id": "number (required)",
    "groups": "[{group_id, role}]",
    "users": "[{user_id, role}]"
  }
  ```

### Alerts

`…/core/watched-groups/`

- **GET** / **POST** `…/watched-groups/` — list / create watched group.
- **GET** / **PATCH** / **DELETE** `…/watched-groups/{id}/` — watched group detail / update / delete. DELETE is destructive — confirm with the user first.
- **GET** / **POST** `…/watched-groups/{watchedGroupPk}/watched-users/` — list / add watched users.
- **DELETE** `…/watched-users/{id}/` — remove watched user. Destructive — confirm with the user first.
- **GET** / **POST** `…/watched-groups/{watchedGroupPk}/watchers/` — list / add watchers.
- **PATCH** / **DELETE** `…/watchers/{id}/` — update / remove watcher (notification-event flags in body). DELETE is destructive — confirm with the user first.

## Example

List the first page of organization users with their policies:

```bash
curl -s \
  "https://api.iblai.app/dm/api/core/platform/users/?platform_key=$IBLAI_ORG&platform_org=$IBLAI_ORG&page=1&page_size=10&return_policies=true" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- Most endpoints require `platform_key={org}` as a query param even on POST/PUT
  bodies that also carry it — send both.
- Promote/demote is the only call on the LMS host (`learn.iblai.app`); every
  other endpoint here is on the DM host under `/api/core/…`.
- Policies bind a `role` (RBAC role id) to `resources`, `users`, and `groups` —
  create the Role first, then reference its id when creating the Policy.
- DELETE on any sub-section is destructive — confirm with the user before
  removing groups, roles, policies, teams, watched groups/users, or watchers.
