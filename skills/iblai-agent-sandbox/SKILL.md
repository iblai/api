---
name: iblai-agent-sandbox
description: Connect/disconnect an ibl.ai agent's sandbox (Claw) instance, manage Claw instances, push config, run health/connectivity checks, and set the agent's model — via the platform API. Use when wiring an agent to a sandboxed workspace.
---

# iblai-agent-sandbox

Wire an agent to a Claw sandbox instance, manage the available instances, push
config and run health/connectivity checks, and set the model the agent's
workspace uses. Use when connecting an agent to a sandboxed workspace.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `…/orgs/{org}/mentors/{mentor}/claw-config/` — connected instance (404 = not connected).
- **GET** `…/orgs/{org}/claw/instances/` — available Claw instances.
- **GET** `…/orgs/{org}/mentors/{mentor}/agent-config/` — agent workspace config (includes current model).
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor-llms/?mentor_id={mentor}` — available model options.

## Writes

- **POST** `…/mentors/{mentor}/claw-config/` — connect to an instance:
  ```json
  {
    "server": "instance id (required)",
    "enabled": "boolean",
    "agent_id": "string"
  }
  ```
- **PATCH** `…/mentors/{mentor}/claw-config/` — toggle auto-push on save:
  ```json
  { "auto_push": "boolean" }
  ```
- **DELETE** `…/mentors/{mentor}/claw-config/` — disconnect. Destructive — confirm with the user first.
- **POST** `…/mentors/{mentor}/claw-config/push-config/` — push config.
- **POST** `…/claw/instances/{id}/health-check/` — run health check.
- **POST** `…/claw/instances/{id}/test-connectivity/` — run connectivity check.
- **POST** `…/claw/instances/{id}/` — create instance:
  ```json
  {
    "name": "string",
    "claw_type": "openclaw | ironclaw",
    "server_url": "string",
    "gateway_token": "string"
  }
  ```
- **PATCH** `…/claw/instances/{id}/` — edit instance (send only changed keys: `name`, `claw_type` `openclaw|ironclaw`, `server_url`, `gateway_token`, …).
- **DELETE** `…/claw/instances/{id}/` — delete instance. Destructive — confirm with the user first.
- **PATCH** `…/mentors/{mentor}/agent-config/` — set model:
  ```json
  { "model": "string" }
  ```

## Example

Connect an agent to a Claw instance and enable it:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/mentors/$MENTOR/claw-config/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"server": "a3f1c9e0-1234-4d56-89ab-cdef01234567", "enabled": true}'
```

## Notes

- A `404` from `claw-config/` means the agent is not connected to a sandbox yet —
  use the connect POST to wire it up.
- Push config after connecting (or enable auto-push on save via the PATCH) so
  the agent's workspace reflects the latest configuration.
- Run **health-check** and **test-connectivity** against an instance id before
  connecting an agent to confirm the sandbox is reachable.
- `claw_type` is either `openclaw` or `ironclaw`; `gateway_token` authenticates the
  instance to its server.
