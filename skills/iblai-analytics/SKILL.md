---
name: iblai-analytics
description: Read ibl.ai analytics via the platform API across agents, content, and users — overview KPIs, users, topics, transcripts, costs, courses, programs, and audit — plus generate and download Data Reports. Scope per-agent or organization-wide. Use to pull usage, engagement, cost, catalog, or per-user learning data.
---

# iblai-analytics

Read ibl.ai analytics from the API across all three scopes:

- **Agent analytics** — include `mentor_unique_id` to scope any metric to one agent.
- **Organization / content analytics** — omit `mentor_unique_id` for org-wide
  metrics, plus the catalog **Courses** and **Programs** breakdowns.
- **User analytics** — a single user's enrollments, grades, time spent, engagement.

Reads are read-only; the only writes are Data Reports.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = an agent's unique id (**optional** — include for agent scope, omit
  for org-wide).
- **Host:** analytics live on **DM** at `…/dm/api/analytics/…`; **Audit** is on
  DM under `…/dm/api/ai-mentor/…`.
- **Chart params:** every chart passes `date_filter` (`today` | `7d` | `30d` |
  `90d` | `custom`; `custom` adds `start_date` / `end_date`), `platform_key={org}`,
  optional `mentor_unique_id={mentor}` (agent scope), and optional `usergroup_ids`.
  Below, `…` = `https://api.iblai.app/dm/api` and a trailing `&platform_key={org}`
  (plus `&mentor_unique_id={mentor}` when scoping to an agent) is implied as `&…`.
- Not connected yet? Run **`/iblai-login`** first.

## Reads — Overview / Users / Topics / Transcripts / Costs

These endpoints serve both **agent** scope and **organization-wide** scope;
include `mentor_unique_id` for the former, omit it for the latter.

### Overview

- **GET** `…/analytics/topics/?date_filter=30d&…` — Messages / Topics / Conversations KPIs.
- **GET** `…/analytics/users/?metric=active_users_last_30d&date_filter=30d&…` — Active Users KPI.
- **GET** `…/analytics/sessions/?date_filter={r}&…` — Sessions line chart.
- **GET** `…/analytics/topics/details/?date_filter={r}&…` — Topics bar chart.

### Users

- **GET** `…/analytics/users/?metric=registered_users&date_filter=30d&…`
- **GET** `…/analytics/users/?metric=active_users&date_filter={r}&…`
- **GET** `…/analytics/users/?metric=currently_active&date_filter=today&…`
- **GET** `…/analytics/time/?date_filter={r}&…` — access-time heatmap.
- **GET** `…/analytics/users/details/?date_filter={r}&page={n}&limit=5&search={q}&…` — user table.

### Topics

- **GET** `…/analytics/conversations?metric=conversations&date_filter={r}&…`
- **GET** `…/analytics/topics/?date_filter=30d&…`
- **GET** `…/analytics/ratings/?date_filter={r}&…`
- **GET** `…/analytics/topics/details/?date_filter={r}&…`

### Transcripts

- **GET** `…/analytics/conversations?metric=headline&…`
- **GET** `…/analytics/messages/?search={user}&topic={topic}&page={n}&limit=20&…` — transcript list.
- **GET** `…/analytics/messages/details/?session_id={id}&…` — one transcript.

### Costs

- **GET** `…/analytics/financial/?metric=weekly_costs&date_filter=all_time&…`
- **GET** `…/analytics/financial/?metric=monthly_costs&date_filter=all_time&…`
- **GET** `…/analytics/financial/?metric=total_costs&all_time=true&…`
- **GET** `…/analytics/financial/?metric=total_costs&date_filter={r}&…` — cost/day chart.
- **GET** `…/analytics/financial/details/?group_by=provider&date_filter={r}&…`
- **GET** `…/analytics/financial/details/?metrics=total_costs,sessions&group_by=username&date_filter={r}&page={n}&limit={l}&search={q}&…`
- **GET** `…/analytics/financial/details/?metric=total_costs&group_by=llm_model&date_filter={r}&…`

## Reads — Content (Courses & Programs)

Course-level and program-level analytics are org-wide catalog breakdowns
(no `mentor_unique_id`) on the same `…/dm/api/analytics/…` family, keyed
by course/program — analogous to the `details` grouping used by Costs (e.g.
`group_by`). Exact `courses` / `programs` sub-paths are platform-specific; use
the `…/analytics/…/details/` endpoints grouped by course/program, or capture the
precise sub-path from the live API responses.

## Reads — User analytics

A single user's own learning data (enrollments, grades, time spent, engagement,
last access). RBAC-gated (`IsPlatformAdminOfUser | IsSelfAccess`).

- **GET** `…/analytics/learners/?username={username}&limit={n}&page={n}&start_date=&end_date=&date_filter={r}` — unified user analytics (user info, summary metrics, per-platform results).
- **GET** `…/analytics/learner/details?username={username}&start_date=&end_date=&date_filter={r}&metrics={…}` — detailed per-course analytics across catalog, agent, and credential data.

(`…/analytics/learners/` and `…/analytics/learner/details` keep the platform's
wire spelling.)

## Audit

- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentors/audit-logs/?limit=20&offset={n}&mentor={mentor}[&action=0|1|2&actor_email=&from_date=&to_date=]`

## Writes (Data Reports)

Data Reports (`…/reports`) are the only writes: kick off a report, poll until
complete, then download the finished CSV from the presigned URL.

- **GET** `…/reports/platforms/{org}/?mentor_id={mentorDbId}` — list reports + status.
- **POST** `…/reports/platforms/{org}/new` — generate a report:
  ```json
  {
    "report_name": "string (e.g. ai-mentor-chat-history)",
    "mentor": "uuid",
    "usergroup_ids": "number[]",
    "start_date": "yyyy-MM-dd",
    "end_date": "yyyy-MM-dd",
    "course_id": "string",
    "query": "string"
  }
  ```
- **GET** `…/reports/platforms/{org}/{report_name}?mentor_unique_id={mentor}` — poll status until complete.
- **GET** `{status.url}` — download the finished CSV.

## Example

Org-wide Messages/Topics/Conversations KPIs (omit `mentor_unique_id`), then the
same scoped to one agent:

```bash
curl -s "https://api.iblai.app/dm/api/analytics/topics/?date_filter=30d&platform_key=$IBLAI_ORG" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"

curl -s "https://api.iblai.app/dm/api/analytics/topics/?date_filter=30d&platform_key=$IBLAI_ORG&mentor_unique_id=$MENTOR" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- Same endpoints serve agent and org/content scope — the presence of
  `mentor_unique_id` is the only difference.
- `date_filter=custom` requires both `start_date` and `end_date` (`yyyy-MM-dd`).
- `usergroup_ids` narrows any chart (and a report) to specific user groups.
- For *finding* agents/content use `/iblai-search`; for recommendations use
  `/iblai-search`.
