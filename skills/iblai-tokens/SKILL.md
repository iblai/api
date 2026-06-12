---
name: iblai-tokens
description: Manage an organization's Platform API Tokens via the platform API — list, create (secret shown once), and delete Api-Tokens by name. Use when issuing or rotating the keys that authenticate ibl.ai API access.
---

# iblai-tokens

Manage the organization's **Platform API Tokens** — the keys that authenticate every
ibl.ai API call. List the tokens, create a new Api-Token (the secret is shown only
once), and delete a token by name. Tokens are `platform_key`-scoped, not
agent-scoped.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- **Host:** these endpoints live under `…/dm/api/core/…`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `https://api.iblai.app/dm/api/core/platform/api-tokens/?platform_key={org}` — list API keys.

## Writes

- **POST** `https://api.iblai.app/dm/api/core/platform/api-tokens/` — create a token (returns the secret only once):
  ```json
  {
    "username": "string (required)",
    "name": "string (required)",
    "key": "",
    "platform_key": "{org} (required)",
    "created": "ISO datetime (required)",
    "expires": "'' or seconds-string (required)",
    "expires_in": "seconds-string | undefined"
  }
  ```
- **DELETE** `https://api.iblai.app/dm/api/core/platform/api-tokens/{name}?platform_key={org}` — delete a key by name. Destructive — confirm with the user first.

## Example

Create a new Platform API Token named `prod-integration` (capture the secret from
the response — it is shown only once):

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/core/platform/api-tokens/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "'"$IBLAI_USERNAME"'",
    "name": "prod-integration",
    "key": "",
    "platform_key": "'"$IBLAI_ORG"'",
    "created": "2026-06-12T00:00:00Z",
    "expires": ""
  }'
```

## Notes

- The create response returns the token secret **only once** — store it
  immediately; it cannot be retrieved again afterward.
- `/iblai-login` uses this same `POST …/platform/api-tokens/` endpoint to mint
  the Api-Token it stores as `IBLAI_API_KEY`.
- Delete is by token **name** (not id), and is scoped to the org via
  `platform_key={org}`.
