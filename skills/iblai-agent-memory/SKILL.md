---
name: iblai-agent-memory
description: Manage an ibl.ai agent's memories via the platform API — list and filter memories (by category, user, email, date), add/edit/delete memories, and manage memory categories. Use when inspecting or curating what an agent remembers.
---

# iblai-agent-memory

Manage an agent's memories through the API: browse and filter what an agent has
remembered, add / edit / delete individual memories, and manage the categories
memories are filed under. Use when inspecting or curating what an agent
remembers.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `…/users/{username}/mentors/{mentor}/mentor-memories-list/?page={n}&page_size={n}&category={slug}&my_memory={bool}&user_id={id}&email={e}&start_date={yyyy-MM-dd}&end_date={yyyy-MM-dd}` — paged memory list.
- **GET** `…/users/{username}/mentors/{mentor}/mentor-memories/?start_date=&end_date=&email=` — memories grouped by category.
- **GET** `…/orgs/{org}/mentors/{mentor}/memory-categories/` — category list.

## Writes

- **POST** `…/mentors/{mentor}/mentor-memories/` — add a memory:
  ```json
  {
    "category_slug": "string (required)",
    "content": "string (required)"
  }
  ```
- **PATCH** `…/mentor-memories/{memoryId}/` — edit a memory:
  ```json
  {
    "category_slug": "string",
    "content": "string"
  }
  ```
- **DELETE** `…/mentor-memories/{memoryId}/` — delete one memory (no body). Destructive — confirm with the user first. Bulk delete = one call per memory.
- **POST** `…/mentors/{mentor}/memory-categories/` — add a category:
  ```json
  {
    "name": "string (required)",
    "slug": "string (required)",
    "description": "string",
    "extraction_prompt": "string",
    "is_active": "boolean"
  }
  ```
- **PATCH** `…/memory-categories/{categoryId}/` — edit a category.
- **DELETE** `…/memory-categories/{categoryId}/` — delete a category (no body). Destructive — confirm with the user first.

## Example

List the first page of memories filed under the `preferences` category since the start of the year:

```bash
curl -s \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentors/$MENTOR/mentor-memories-list/?page=1&page_size=20&category=preferences&start_date=2026-01-01" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- Memories are filed under categories; `category_slug` on a memory must match an
  existing category's `slug` from the categories endpoint.
- The `mentor-memories-list/` filters stack — combine `category`, `user_id`,
  `email`, and the date range to narrow results.
- There is no bulk-delete endpoint: to clear several memories, issue one DELETE
  per `memoryId`.
