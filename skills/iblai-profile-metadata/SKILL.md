---
name: iblai-profile-metadata
description: Read and write per-user, per-organization metadata via the platform API — a key-value store for user preferences, app settings, feature flags, and onboarding progress. Defaults to the signed-in user; admins can target another user. Use to persist per-user state.
---

# iblai-profile-metadata

Read and write per-user, per-organization metadata via the API: a key-value store
for preferences, app settings, feature flags, and onboarding progress, all served
from a single `…/dm/api/core/users/platform-metadata/` endpoint. Defaults to the
signed-in user; admins can target another user.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG` (passed as the `platform_key` query
  param), `{username}` = `$IBLAI_USERNAME`.
- **Host:** all operations hit `…/dm/api/core/users/platform-metadata/` and take
  `?platform_key={org}`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Endpoints (all take `?platform_key={org}`; admins may add `&username={other_user}`)

- **GET** `https://api.iblai.app/dm/api/core/users/platform-metadata/?platform_key={org}` — retrieve all metadata (defaults to the authenticated user).
- **PATCH** `https://api.iblai.app/dm/api/core/users/platform-metadata/?platform_key={org}` — update or delete specific keys without touching others (include at least one of `metadata` or `delete_keys`):
  ```json
  {
    "metadata": { "<key>": "<value>" },
    "delete_keys": ["<key>"]
  }
  ```
- **PUT** `https://api.iblai.app/dm/api/core/users/platform-metadata/?platform_key={org}` — replace ALL metadata:
  ```json
  {
    "metadata": { }
  }
  ```
- **DELETE** `https://api.iblai.app/dm/api/core/users/platform-metadata/?platform_key={org}` — clear all metadata for the user. Destructive — confirm with the user first.

## Admin (target another user)

- Append `&username={other_user}` to any of the above (admin only) to
  read/update/replace/delete that user's metadata.

## Example

Persist onboarding progress and a theme preference for the signed-in user, leaving
all other keys untouched:

```bash
curl -X PATCH \
  "https://api.iblai.app/dm/api/core/users/platform-metadata/?platform_key=$IBLAI_ORG" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "metadata": { "onboarding_step": "3", "theme": "dark" } }'
```

## Notes

- Metadata is scoped per user **and** per organization (`platform_key`) — the same
  user can hold different metadata in different orgs.
- A PATCH with neither `metadata` nor `delete_keys` returns `400`.
- PATCH is the safe choice for incremental updates; PUT replaces the entire object,
  so any key not included is dropped.
