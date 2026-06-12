---
name: iblai-agent-create
description: Create a new ibl.ai agent from a template via the platform API — set name, display name, description, system prompt, and LLM provider; returns the new agent's unique id. Use when creating an agent from scratch, then configure it with the other /iblai-agent-* skills.
---

# iblai-agent-create

Create a brand-new agent from a template. This is the step **before**
configuring an agent: it mints a new agent and returns its `unique_id`, which
you then hand to the `/iblai-agent-*` skills (settings, llm, prompts,
datasets, …) to configure further. To duplicate an existing agent instead of
starting from a template, use `/iblai-agent-settings` (fork).

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `https://api.iblai.app/dm/api/search/orgs/{org}/users/{username}/mentors/` — list existing agents in the org (useful to pick a fork source or confirm names are free).
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor/categories/` — category options (to assign after creation via `/iblai-agent-settings`).

## Writes

- **POST** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor-with-settings/` — create a new agent from a template (returns the created agent with its `unique_id`):
  ```json
  {
    "template_name": "string (required, e.g. ai-mentor)",
    "new_mentor_name": "string (required)",
    "display_name": "string",
    "description": "string",
    "system_prompt": "string",
    "llm_provider": "string"
  }
  ```

## Example

Create a data-science agent from the default `ai-mentor` template, then capture
its `unique_id` for follow-up configuration:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentor-with-settings/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "template_name": "ai-mentor",
    "new_mentor_name": "Data Science Tutor",
    "description": "Helps users with statistics and Python."
  }'
# → response includes "unique_id"; use it as {mentor} in the other skills
```

## Notes

- `template_name` defaults to `ai-mentor` on most orgs; the create call accepts
  any template the org exposes.
- Creation is intentionally minimal — set only name + a couple of fields here,
  then drive the rest with the focused skills: `/iblai-agent-settings`,
  `/iblai-agent-llm`, `/iblai-agent-prompts`, `/iblai-agent-datasets`,
  `/iblai-agent-tools`, etc., using the returned `unique_id` as `{mentor}`.
- To copy an existing agent rather than start from a template, use the **fork**
  endpoint in `/iblai-agent-settings`.
