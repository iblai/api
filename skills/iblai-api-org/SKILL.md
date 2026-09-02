---
name: iblai-api-org
description: Read and write an ibl.ai organization's org-wide settings via the platform API — the org metadata object holding Default Agent, Help Center URL, Chat Area Width, and feature toggles (Help/Accessibility menus, Community Agents, Report Inappropriate Content) — plus the org's custom domains (list, create, point a domain at an SPA, soft-delete/restore, hard delete). Use when configuring org-wide behavior or wiring a vanity domain to an org.
---

# iblai-api-org

Read and write an organization's org-wide settings from the API. Two surfaces:

- **Org settings** — Default Agent, Help Center URL, Chat Area Width, and the
  feature toggles. These all live inside one **metadata** object.
- **Custom domains** — the vanity hostnames pointed at this org, and which SPA
  each one serves.

Use when configuring org-wide behavior or attaching a domain to an org.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- **Every setting is a key inside ONE org-metadata object.** To change a
  setting you **GET** the whole object, **merge** your changed key into it, and
  **PUT** the whole object back. Never drop existing keys — anything you omit is
  lost.
- **Custom domains take the org key as a `platform_key` query/body field**, not
  as a path segment — the value is still `$IBLAI_ORG`.
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.
- Deleting a custom domain is destructive and outward-facing — **confirm with
  the user first.**

## Concepts — custom domains

A custom domain row binds one hostname to one org **and one SPA**. `spa` is a
fixed enum — `auth` · `skillsai` · `mentorai` · `analyticsai` (wire values, kept
verbatim) — defaulting to `auth`. One hostname maps to exactly one SPA, so
serving several apps on vanity domains means several rows.

Two deletes exist and they are not the same:

- **Soft delete** flips `is_deleted`, hiding the domain from the default listing
  while keeping the row. It is reversible — send `is_deleted: false` to restore.
- **Hard delete** removes the row permanently. Not reversible.

`registered_with_dns_pro` is **not writable**. DNS/SSL provisioning is driven by
a backend signal after the row is created; the flag reports that outcome and the
`instructions` field carries the DNS setup steps to hand the operator. Creating a
domain does not make it resolve — DNS still has to be pointed.

## Reads

### Org settings

- **GET** `https://api.iblai.app/dm/api/core/orgs/{org}/metadata/` — read all org settings.
- **GET** `https://api.iblai.app/dm/api/search/orgs/{org}/users/{username}/mentors/` — source of valid Default Agent values.

### Custom domains

- **GET** `https://api.iblai.app/dm/api/custom-domains/?platform_key={org}` — list this org's domains. **Public endpoint — no auth required.** You must pass **either** `platform_key` **or** `domain` (a bare hostname lookup); sending neither is a `400`. Optional filters:
  - `?include_deleted=true` — include soft-deleted rows (default excludes them).
  - `?status=` — filter on DNS registration: `true` / `1` / `active` / `registered`, or `false` / `0` / `inactive` / `unregistered`.

  Returns `{"custom_domains": [...], "count": n}`, or an empty object `{}` when the org has none — **not** a `404`. An unknown `platform_key` *is* a `404`.

  Each domain object: `id`, `custom_domain`, `spa`, `spa_display`, `registered_with_dns_pro`, `dns_pro_display`, `instructions`, `is_deleted`, `platform_id`, `platform_key`, `platform_name`, `platform_metadata`, `created_at`, `updated_at`.

## Writes

### Org settings

- **PUT** `https://api.iblai.app/dm/api/core/orgs/{org}/metadata/` — save settings (send the whole object back with your key merged in):
  ```json
  {
    "metadata": {
      "…existing…": "preserved",
      "<settingKey>": "boolean|string"
    }
  }
  ```
  - **Default Agent** → `overall_default_mentor` (agent `unique_id` or `"none"`)
  - **Help Center URL** → `help_center_url` (string)
  - **Chat Area Width** → `chat_area_size` (string)
  - **Help Menu** / **Accessibility Menu** / **Persistent Chat Input Label** /
    **Community Agents** / **Report Inappropriate Content** → boolean, keyed by
    the metadata-catalog slug assigned at runtime.

### Custom domains

- **POST** `https://api.iblai.app/dm/api/custom-domains/create/` — attach a domain to the org. Requires org admin.
  ```json
  {
    "platform_key": "$IBLAI_ORG",
    "custom_domain": "learn.example.com",
    "spa": "mentorai"
  }
  ```
  - `platform_key` and `custom_domain` **required** — omitting either is a `400`.
  - `spa` optional, defaults to `auth`; an unrecognized value is a `400` listing the valid choices.
  - `is_deleted` optional (create pre-soft-deleted; rarely useful).
  - `201` → `{"message": …, "custom_domain": {…}}`. A hostname already in use anywhere on the platform is a `400` ("Domain already exists") — hostnames are globally unique, not per-org.
- **PUT** `…/custom-domains/{domain_id}/status/` — repoint an existing domain at a different SPA: `{"spa": "skillsai"}`. `spa` is required; invalid values are `400`, unknown `domain_id` is `404`.
- **PUT** `…/custom-domains/by-name/{domain_name}/status/` — the same change addressed by hostname instead of id, for when you know the domain but not its row id. Requires org admin.
- **POST** `…/custom-domains/{domain_id}/deleted-status/` — soft delete or restore: `{"is_deleted": true}` hides it, `{"is_deleted": false}` brings it back. The field is required; omitting it is a `400`. **Confirm with the user first.**
- **DELETE** `…/custom-domains/{domain_id}/delete/` — **hard** delete, permanent and irreversible. Returns the deleted row for the record. Prefer the soft delete above unless the user explicitly wants the row gone. **Confirm with the user first.**

## Example

Set the Help Center URL while preserving every other key — read first, merge,
then write the whole object back:

```bash
# 1. read the current metadata object
curl -s "https://api.iblai.app/dm/api/core/orgs/$IBLAI_ORG/metadata/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" > metadata.json

# 2. merge the changed key into the object, then PUT it all back
curl -X PUT \
  "https://api.iblai.app/dm/api/core/orgs/$IBLAI_ORG/metadata/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"…existing keys preserved…": "...", "help_center_url": "https://help.example.com"}}'
```

Point a vanity domain at the agent SPA, then read back the DNS instructions:

```bash
# 1. attach the domain to this org
curl -X POST "https://api.iblai.app/dm/api/custom-domains/create/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"platform_key\":\"$IBLAI_ORG\",\"custom_domain\":\"learn.example.com\",\"spa\":\"mentorai\"}"

# 2. list the org's domains and read `instructions` for the DNS records to set
curl -s "https://api.iblai.app/dm/api/custom-domains/?platform_key=$IBLAI_ORG"
```

## Notes

- The PUT replaces the metadata object — always GET first, merge your one key,
  and resend the full object so existing settings survive.
- `overall_default_mentor` takes a agent `unique_id` from the mentor-source
  endpoint, or the literal string `"none"` to clear it.
- The toggle keys (Help Menu, Accessibility Menu, Persistent Chat Input Label,
  Community Agents, Report Inappropriate Content) are slugs assigned by the
  metadata catalog at runtime — read them off the GET response rather than
  hardcoding.
- **Custom-domain reads are public; writes are not.** The list endpoint takes no
  auth at all, so it is safe to call before `/iblai-api-login`. Every write needs
  an admin token.
- **An empty list comes back as `{}`, not `{"custom_domains": [], "count": 0}`** —
  read defensively rather than indexing straight into `custom_domains`.
- **Hostnames are globally unique.** "Domain already exists" on create can mean
  another org holds it, not just this one — the error does not distinguish.
- **Creating a row does not provision DNS or TLS.** That happens asynchronously
  after create; poll `registered_with_dns_pro` (via the list endpoint) to see
  whether it landed, and hand `instructions` to whoever controls the zone.
