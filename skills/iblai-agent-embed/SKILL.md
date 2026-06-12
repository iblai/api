---
name: iblai-agent-embed
description: Configure an ibl.ai agent's embed/widget settings via the platform API — anonymous access, custom CSS/JS, mode, voice/attachment toggles, starter prompts, SSO, share links, and redirect tokens. Use when embedding an agent on an external website.
---

# iblai-agent-embed

Configure an agent's embed/widget settings through the API: anonymous access,
custom CSS/JS, widget mode, voice/attachment toggles, starter prompts, SSO, plus
the share link and redirect tokens that let an agent live on an external site.
Use when embedding an agent on an external website.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- **Embed writes** go through one endpoint —
  **PUT** `…/mentors/{mentor}/settings/` with `multipart/form-data` — sending
  **only the changed field(s)**.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `…/mentors/{mentor}/public-settings/` — current embed settings.
- **GET** `…/mentors/{mentor}/settings/` — current custom CSS/JS.
- **GET** `…/mentors/{mentor}/sharable-link` — existing share-link token (404 = none).
- **GET** `https://learn.iblai.app/ibl-auth/get-provider-slug/?platform_key={org}&username={username}` — SSO providers (LMS host).

## Writes

- **PUT** `…/mentors/{mentor}/settings/` — save embed settings (`multipart/form-data`, send only changed keys):
  ```json
  {
    "allow_anonymous": "boolean",
    "custom_css": "string",
    "website_url": "string",
    "mode": "default|advanced",
    "is_context_aware": "boolean",
    "mentor_visibility": "string|null",
    "embed_show_attachment": "boolean",
    "embed_show_voice_call": "boolean",
    "embed_show_voice_record": "boolean",
    "show_catalogue": "boolean",
    "starter_prompts": "guided_prompt|suggested_prompt",
    "sso": "boolean",
    "sso_provider": "string",
    "auto_open": "boolean",
    "slug": "string",
    "safety_disclaimer": "boolean"
  }
  ```
  To set advanced styling, send just `custom_css`; for advanced scripting, send `custom_javascript`.
- **POST** `…/mentors/{mentor}/sharable-link` — create / regenerate the share link (empty body).
- **PUT** `…/mentors/{mentor}/sharable-link` — enable / disable the share link:
  ```json
  {
    "enabled": "boolean"
  }
  ```
- **POST** `https://api.iblai.app/dm/api/core/orgs/{org}/redirect-tokens/` — mint a redirect token for the embed site:
  ```json
  {
    "url": "string (required)",
    "mentor_unique_id": "string"
  }
  ```

## Backend token provisioning (server-to-server embed auth)

Skip the SSO login popup by minting the user's tokens from your own backend, then
handing the embed the same `ibl-data` blob the Auth SPA would have produced. All
calls are **server-to-server** with the org's Platform API Token — never expose
this in browser JS.

- **POST** `https://api.iblai.app/dm/api/core/consolidated-token/provision/` — resolve-or-create the user and mint `dm_token` + `axd_token`.
- **GET** `https://api.iblai.app/dm/api/core/users/platforms/` — fetch the user's organization/platform record.
- Assemble those into the `ibl-data` payload and pass it to the iframe.

Gated per-org by the ibl.ai-managed flag `ENABLE_PLATFORM_CONSOLIDATED_PROXY_PROVISIONING`
(must be `true`, else the provision endpoint returns **404** — contact ibl.ai to enable).

## Example

Allow anonymous access and turn on the widget's attachment button (only the two
changed fields are sent):

```bash
curl -X PUT \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentors/$MENTOR/settings/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -F "allow_anonymous=true" \
  -F "embed_show_attachment=true"
```

## Notes

- A field left out of the PUT is left unchanged — never resend the whole object.
- The SSO providers read uses the LMS host `learn.iblai.app`, not `api.iblai.app`.
- A `404` on **GET** `…/sharable-link` just means no share link exists yet —
  **POST** to create one.
