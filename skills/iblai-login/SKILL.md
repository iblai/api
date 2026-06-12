---
name: iblai-login
description: Connect an ibl.ai organization for API access. Opens login.iblai.app/me in the browser so the user signs in via SSO, captures their org key and username, helps mint a Platform API Token, and writes IBLAI_ORG / IBLAI_USERNAME / IBLAI_API_KEY to .env. Run this first before any other iblai-* skill.
---

# iblai-login

Connect an organization so the other `iblai-*` skills can call the ibl.ai platform.
Every API call needs three things — your **org key**, your **username**, and a
**Platform API Token** — and this skill collects all three from
[login.iblai.app/me](https://login.iblai.app/me) and writes them to `.env`.

Run this once per organization. After it succeeds, every other skill reads the values
from `.env`.

## What you are collecting

| Value             | Goes in `.env` as    | Where it comes from                                   |
| ----------------- | -------------------- | ----------------------------------------------------- |
| Org key  | `IBLAI_ORG`          | `login.iblai.app/me` → the **key** of the chosen org  |
| Username          | `IBLAI_USERNAME`     | `login.iblai.app/me` → account / profile              |
| Platform API Token| `IBLAI_API_KEY`      | An Api-Token minted for the org — see step 3          |

The base URL is fixed: `https://api.iblai.app`. Auth header on every request is
`Authorization: Api-Token <IBLAI_API_KEY>`.

## Steps

> **Tip — use browser automation.** This skill is smoothest when the agent can
> drive a browser (e.g. the user launches **`claude --chrome`**): it can open
> `login.iblai.app/me`, read the signed-in session, and **mint the API token
> automatically** (step 3). Without a browser you'll walk the user through the
> page and they paste a token instead — recommend they relaunch with
> `claude --chrome` for the automated path.

1. **Open the account page in the browser.**

   Navigate the user to **https://login.iblai.app/me**.

   **If the user is not logged in** (the page redirects to a sign-in screen
   instead of showing "My Account"), do not try to authenticate for them. Print
   the login URL and ask them to sign in, then continue once they confirm:

   > You are not signed in to ibl.ai. Open this URL, log in (email, Apple,
   > Google, or password — SSO/OAuth/OIDC/SAML where your org configures it),
   > then tell me when you are done:
   >
   > **https://login.iblai.app/me**

   Wait for the user to confirm they have logged in. **After login the platform
   redirects somewhere else (the destination varies and may change), not back to
   `/me`** — so always **re-navigate explicitly to `https://login.iblai.app/me`**
   before reading. Do not assume you are already on `/me`; the only thing that
   matters is that once signed in you can reach `login.iblai.app/me`.

   > Do not enter the user's credentials for them — let them complete the login
   > themselves. Only read the values off the page once they are signed in.

2. **Read the org key and username off `/me`.**

   `/me` is server-rendered (no separate JSON API to call), so read the values
   off the **rendered page content**. It shows the account (username/email) and
   the list of organizations the user belongs to; each org block is the display
   name followed by its **key** — e.g. `enterprise`, `main`, a company slug, or a
   UUID like `3b42a400a2fc4ec9...`. Accounts can belong to many orgs (40+ is
   normal), so **always ask the user which org to target** rather than picking
   one — capture that org key → `IBLAI_ORG`, and the account username →
   `IBLAI_USERNAME`.

   **If the only org is `main`:** `main` is the shared default everyone lands in
   — it is *not* the user's own workspace. If `/me` shows only `main`, the user
   must **create their own organization** before continuing.
   > Org-creation endpoint: _to be added._ Until then, direct the user to create
   > an organization (in the platform / at `login.iblai.app`), then re-read `/me`
   > and pick the new org.

3. **Mint a Platform API Token from the session.**

   API calls authenticate with an **Api-Token**, not the browser login. Create
   one for the chosen org:

   - **With browser automation (recommended):** once the user is signed in on
     `login.iblai.app/me`, the page's session carries a short-lived auth token —
     read **`dm_token`** from that site's `localStorage`. Use it as a Bearer token
     to create a Platform API Token (the same `POST …/platform/api-tokens/` call
     documented in **`/iblai-tokens`**):

     ```http
     POST https://api.iblai.app/dm/api/core/platform/api-tokens/
     Authorization: Bearer <dm_token from login.iblai.app localStorage>
     Content-Type: application/json

     { "username": "<username>", "name": "iblai-cli", "key": "",
       "platform_key": "<org>", "created": "<ISO-now>", "expires": "" }
     ```

     Capture the returned secret (shown once) → `IBLAI_API_KEY`.

   - **Without a browser:** have the user create a token in the platform admin
     (or at `login.iblai.app`) and paste the secret. You can recommend they
     relaunch with `claude --chrome` to automate this.

   Capture the token → `IBLAI_API_KEY`.

4. **Save to `.env` (and make sure it's gitignored).**

   Once you have the org key + token, create/update `.env` at the project root:

   ```dotenv
   IBLAI_ORG=<org key>
   IBLAI_USERNAME=<username>
   IBLAI_API_KEY=<token>
   ```

   **Before writing it, guarantee `.env` is ignored by git** — check `.gitignore`
   and add a `.env` line if it is not already there, so the token can never be
   committed. Create `.gitignore` with that line if the project has none.

   Every other `iblai-*` skill reads these values. To make them live in shell
   commands, `source .env` (`set -a; . ./.env; set +a`) before API calls — or, in
   Claude Code, mirror them into `.claude/settings.local.json` `env` (also
   gitignored) for automatic injection.

5. **Verify the connection.**

   Confirm the token resolves the org:

   ```bash
   curl -s "https://api.iblai.app/dm/api/core/platform/api-tokens/?platform_key=$IBLAI_ORG" \
     -H "Authorization: Api-Token $IBLAI_API_KEY" | head
   ```

   A `200` with the token list means you are connected. Report the active org and
   username back to the user.

## Notes

- `{org}` (a.k.a. `platform_key`), `{username}`, and `{mentor}` (agent unique id,
  e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`) are the path variables every other
  skill substitutes.
- One organization = one org key + one Api-Token. To switch organizations, re-run this skill
  and pick a different org on `/me`.
- If the user has **no account**, send them to https://ibl.ai/join to sign up —
  that flow leads them through creating their own organization — then restart
  this skill. (An existing account that only has the shared `main` org still
  needs its own org created; see step 2.)
