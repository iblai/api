---
name: iblai-crm
description: Manage an ibl.ai organization's CRM via the platform API — people, organizations, pipelines and stages, lead sources, deals (with stage-move/won/lost actions), activities, and tags. Org-wide sales/relationship management. Use for lead capture, pipelines, and deal flow.
---

# iblai-crm

Manage an organization's **CRM** from the API: people and organizations,
pipelines and their stages, lead sources, deals (with stage-move / won / lost
actions), activities, and tags — all the sales and relationship-management
records for one organization. Use for lead capture, pipelines, and deal flow.

## Auth & conventions

- **Base URL:** `https://platform.iblai.app/api/crm/`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request. (The CRM
  developer docs phrase this as `Authorization: Token <key>` — it is the same
  Platform API Token.)
- **Scope:** Platform-scoped. Every record belongs to the organization resolved
  from the token, so there is **no `{org}` in the path**.
- **Ids:** integers for every resource **except** Person and Organization, whose
  ids are **UUID** strings.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Resources

| Resource     | Path                                      | Id                              |
| ------------ | ----------------------------------------- | ------------------------------- |
| Person       | `/api/crm/persons/`                       | UUID                            |
| Organization | `/api/crm/organizations/`                 | UUID                            |
| Pipeline     | `/api/crm/pipelines/`                      | int                             |
| Stage        | `/api/crm/pipelines/{pipeline_id}/stages/` | int (nested under a pipeline)  |
| Lead Source  | `/api/crm/lead-sources/`                  | int                             |
| Deal         | `/api/crm/deals/`                         | int                             |
| Activity     | `/api/crm/activities/`                    | int                             |
| Tag          | `/api/crm/tags/`                          | int                             |

Each resource supports standard REST: **GET** (list), **POST** (create),
**GET** `{id}` (read), **PATCH** `{id}` (update), **DELETE** `{id}` (delete).
DELETE is destructive — confirm with the user first.

### Person — `/api/crm/persons/` (UUID)

- **GET** `/api/crm/persons/` — list people.
- **POST** `/api/crm/persons/` — create a person.
- **GET** `/api/crm/persons/{id}/` — read a person.
- **PATCH** `/api/crm/persons/{id}/` — update a person.
- **DELETE** `/api/crm/persons/{id}/` — delete a person. Confirm with the user first.

### Organization — `/api/crm/organizations/` (UUID)

- **GET** `/api/crm/organizations/` — list organizations.
- **POST** `/api/crm/organizations/` — create an organization.
- **GET** `/api/crm/organizations/{id}/` — read an organization.
- **PATCH** `/api/crm/organizations/{id}/` — update an organization.
- **DELETE** `/api/crm/organizations/{id}/` — delete an organization. Confirm with the user first.

### Pipeline — `/api/crm/pipelines/` (int)

- **GET** `/api/crm/pipelines/` — list pipelines.
- **POST** `/api/crm/pipelines/` — create a pipeline.
- **GET** `/api/crm/pipelines/{id}/` — read a pipeline.
- **PATCH** `/api/crm/pipelines/{id}/` — update a pipeline.
- **DELETE** `/api/crm/pipelines/{id}/` — delete a pipeline. Confirm with the user first.

### Stage — `/api/crm/pipelines/{pipeline_id}/stages/` (int, nested under a pipeline)

- **GET** `/api/crm/pipelines/{pipeline_id}/stages/` — list a pipeline's stages.
- **POST** `/api/crm/pipelines/{pipeline_id}/stages/` — create a stage.
- **GET** `/api/crm/pipelines/{pipeline_id}/stages/{id}/` — read a stage.
- **PATCH** `/api/crm/pipelines/{pipeline_id}/stages/{id}/` — update a stage.
- **DELETE** `/api/crm/pipelines/{pipeline_id}/stages/{id}/` — delete a stage. Confirm with the user first.

### Lead Source — `/api/crm/lead-sources/` (int)

- **GET** `/api/crm/lead-sources/` — list lead sources.
- **POST** `/api/crm/lead-sources/` — create a lead source.
- **GET** `/api/crm/lead-sources/{id}/` — read a lead source.
- **PATCH** `/api/crm/lead-sources/{id}/` — update a lead source.
- **DELETE** `/api/crm/lead-sources/{id}/` — delete a lead source. Confirm with the user first.

### Deal — `/api/crm/deals/` (int)

- **GET** `/api/crm/deals/` — list deals.
- **POST** `/api/crm/deals/` — create a deal.
- **GET** `/api/crm/deals/{id}/` — read a deal.
- **PATCH** `/api/crm/deals/{id}/` — update a deal.
- **DELETE** `/api/crm/deals/{id}/` — delete a deal. Confirm with the user first.

**Deal actions** (the canonical way to transition deals):

- **PATCH** `/api/crm/deals/{id}/` — reposition `stage` within a pipeline (allowed).
- **POST** `/api/crm/deals/{id}/move-stage/` — transition stage; body accepts
  `stage_code` (preferred) or `stage_id`.
- **POST** `/api/crm/deals/{id}/won/` — close the deal as won.
- **POST** `/api/crm/deals/{id}/lost/` — close the deal as lost.

### Activity — `/api/crm/activities/` (int)

- **GET** `/api/crm/activities/` — list activities.
- **POST** `/api/crm/activities/` — create an activity.
- **GET** `/api/crm/activities/{id}/` — read an activity.
- **PATCH** `/api/crm/activities/{id}/` — update an activity.
- **DELETE** `/api/crm/activities/{id}/` — delete an activity. Confirm with the user first.

### Tag — `/api/crm/tags/` (int)

- **GET** `/api/crm/tags/` — list tags.
- **POST** `/api/crm/tags/` — create a tag.
- **GET** `/api/crm/tags/{id}/` — read a tag.
- **PATCH** `/api/crm/tags/{id}/` — update a tag.
- **DELETE** `/api/crm/tags/{id}/` — delete a tag. Confirm with the user first.

### Useful filters

- `owner={user_id}` — "my pipeline" / "my accounts".
- `owner__isnull=true` — unowned records.
- `?is_default=true` on pipelines — the seeded pipeline; its response embeds its
  stages inline.

## Example

Create a person:

```bash
curl -X POST \
  "https://platform.iblai.app/api/crm/persons/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "Ada",
    "last_name": "Lovelace",
    "email": "ada@example.com",
    "lifecycle_stage": "lead"
  }'
```

## Notes

- The CRM lives on a **different host** than the other `iblai-*` skills:
  `https://platform.iblai.app/api/crm/`, not `https://api.iblai.app`.
- Lifecycle stages are `lead | qualified | opportunity | customer | churned`.
- A CRM Person auto-links to a Platform user when a signup matches by email.
- Stages are nested under a pipeline; deal transitions go through the deal
  actions (`move-stage/`, `won/`, `lost/`) rather than ad-hoc edits.
