---
name: iblai-search
description: Discover agents and learning content in an ibl.ai organization via the platform API — faceted, paginated search over agents and the catalog (courses, programs, pathways, skills), plus personalized (RAG) recommendations for the signed-in user. Use to find, browse, or get recommended agents/content. Read-only.
---

# iblai-search

Discover agents and learning content from the API in three ways: **agent**
search (faceted, full-text search over agents), **content** search
(faceted search over the catalog of courses, programs, pathways, and skills),
and **recommendations** (personalized, RAG-ranked results for the signed-in
user). All read-only. To *edit* an agent use the `/iblai-agent-*` skills.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME` (agent
  search only; content search is not org-pathed — the org comes from the token).
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Agent search

- **GET** `https://api.iblai.app/dm/api/search/orgs/{org}/users/{username}/mentors/` — search/browse agents. Query params:
  - `query` — full-text over name/description (e.g. `calculus`).
  - `category` — numeric category id (from `facets.Category`).
  - `created_by` — author username (from `facets["Created By"]`).
  - `limit`, `page` — pagination.

**Envelope:** `{ results[], count, next, previous, current_page, total_pages, facets }`.
Each result has `unique_id`, `name`, `description`, `llm_provider`, `llm_name`,
`categories`, `created_by`, `is_featured`, `mentor_visibility`, `slug`,
`profile_image`, `settings`. `facets` keys: Category, Created By, Featured, LLM
Provider, Subject, Audience, Promotion, Recently Accessed.

## Content search

- **GET** `https://api.iblai.app/dm/api/search/catalog/` — search/browse the catalog. Query params:
  - `query` — full-text (e.g. `manufacturing`).
  - `limit`, `page` — pagination.
  - Filter params mirror the response `facets`: content
    **type** (course / program / pathway / skill), **language**, **level**,
    **subject**, **format**, **price**, **certificate**.

**Envelope:** `{ results[], count, next, previous, current_page, total_pages, facets }`,
where each result is `{ "type": "course|program|pathway|skill", "data": { } }`.

## Recommendations

Personalized, RAG-ranked results for the signed-in user. Unlike the two
searches above, there is **no query**: results are computed server-side and are
**user-dependent** (personalized to whoever the token resolves to). The org and
user are derived from the token; this endpoint is not org-pathed.

- **GET** `https://api.iblai.app/dm/api/ai-search/recommendations/` — RAG recommendations. Query params:
  - `recommendation_type` — `mentors` | `courses` | `programs` | `resources` | `pathways`.
  - `limit` — maximum number of recommendations.

Returns a ranked list of the requested type. Call once per type for a mixed set.

## Example

Find calculus agents, then manufacturing courses:

```bash
curl -s "https://api.iblai.app/dm/api/search/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentors/?query=calculus&limit=10" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"

curl -s "https://api.iblai.app/dm/api/search/catalog/?query=manufacturing&limit=10" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- Read-only discovery — no writes on either surface.
- The `facets` block in each response enumerates the available filter dimensions
  and counts; use its keys to drive additional filter params.
- An agent result's `unique_id` is the `{mentor}` you pass to the
  `/iblai-agent-*` skills; a content result's `data` shape depends on its
  `type`.
- **Search vs recommendations:** search takes an explicit `query` + filters and
  is the same for any caller; recommendations take no query and are personalized
  per user (token-dependent).
