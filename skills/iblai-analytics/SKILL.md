---
name: iblai-analytics
description: Read ibl.ai analytics via the platform API — overview KPIs, users, topics, transcripts, costs, and catalog performance for courses, programs, pathways, and skills — plus per-learner drill-downs, time-on-platform, and async Data Reports (generate, poll, download). Scope per-agent or platform-wide. Use to pull usage, engagement, cost, catalog, or per-user learning data.
---

# iblai-analytics

Read ibl.ai analytics and generate Data Reports from the platform API. Every endpoint lives under `https://api.iblai.app/dm/api/analytics/…` or `https://api.iblai.app/dm/api/reports/platforms/{platform_key}/…`.

Reads are read-only. Reports are the only writes: kick one off, poll until it finishes, download the CSV or JSON.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{platform_key}` = `$IBLAI_ORG` (the platform key on the wire — Reports uses this).
- **Anchor:** `{dm_url}` = `https://api.iblai.app/dm` throughout the skill. Every URL below is written `{dm_url}/api/analytics/…` or `{dm_url}/api/reports/…` and substitutes to the full https URL at call time. Matches the anchor used in vibe's iblai-* skills.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG` and `IBLAI_API_KEY`.
- Confirm with the user before creating a Data Report — the job runs against the platform and its output may be shared.

## Ground source: OpenAPI schema (priority)

**The OpenAPI schema is the contract, not this file.** Every URL, parameter, enum, and response shape below is orientation only — consult the schema before writing any request. If this file disagrees with the schema, the schema wins.

Two sources, in priority order:

1. **Live schema (authoritative)** — `https://api.iblai.app/dm/api/docs/schema/?format=json`. Browse at `https://api.iblai.app/dm/api/docs/`.
2. **Local snapshot (offline bootstrap)** — [`references/analytics-schema.json`](references/analytics-schema.json). Filtered to `/api/analytics/*` and `/api/reports/*`. Refresh with the routine in [`references/schema.md`](references/schema.md). Reference only; the live schema wins on any disagreement.

See [`references/schema.md`](references/schema.md) for fetch, filter, refresh, and drift-check commands. Run the drift check before shipping any code that hits these endpoints — every URL cited here must appear in the live schema.

## Concepts

### Scope: agent vs platform

Every `/api/analytics/*` endpoint accepts an optional `mentor_unique_id` query param. Include it to scope a metric to one agent; omit it for platform-wide. Same endpoints, same shapes — one param toggles scope.

### Common query parameters

| Parameter | Meaning |
|---|---|
| `date_filter` | `today` \| `7d` \| `30d` \| `90d` \| `all_time` \| `custom`. With `custom`, also send `start_date` and `end_date` (`yyyy-MM-dd`). |
| `platform_key` | Optional on most endpoints; **required** on `learners/list/`. Defaults to the platform on the token. |
| `mentor_unique_id` | Optional. Agent scope when present. |
| `usergroup_ids` | Optional. Narrows any read (and any Report) to specific user groups. |
| `granularity` | `day` \| `week` \| `month` (where the endpoint supports timeseries). |
| `page`, `limit` | Pagination for listing endpoints. |
| `search` | Free-text filter (users, topics, transcripts). |

### RBAC

Every read requires the `Ibl.Analytics/CanViewAnalytics/action` role (page-access gate) **plus** one of:

- `Ibl.Analytics/Core/read` — user or usergroup analytics.
- `Ibl.Analytics/Mentors/read` — mentor analytics (grant on a usergroup or user to row-filter to specific users).

Mentor analytics pages also require `Ibl.Analytics/CanViewMentorAnalytics/action`. Reports also require `Ibl.Analytics/Reports/read`. A learner may read their own analytics without the page gate.

### Data Reports lifecycle

Reports run asynchronously. `POST /new` returns a `report_id` and a `state`. Poll the detail endpoint until `state` is `completed`, then hit `download`. Terminal states: `completed` (download at `url`), `cancelled`, `error`, `expired` (output evicted — regenerate). Intermediate states: `pending → running → accumulating → processing → storing`.

## Reads

Every URL below sits under `{dm_url}/api/analytics/`. All accept `platform_key`, `mentor_unique_id`, `usergroup_ids`, `date_filter`, `start_date`, `end_date` unless noted. Query strings elide trailing repeated params for brevity.

### Overview

Headline KPIs for the landing page.

| Intent | Endpoint |
|---|---|
| Messages / topics KPIs | **GET** `{dm_url}/api/analytics/topics/?date_filter=30d` |
| Conversations chart | **GET** `{dm_url}/api/analytics/conversations/?date_filter=30d` |
| Active users headline | **GET** `{dm_url}/api/analytics/users/?metric=active_users_last_30d&date_filter=30d` |
| Sessions timeseries | **GET** `{dm_url}/api/analytics/sessions/?date_filter=30d` |

`topics/` and `sessions/` accept `metric` to switch between the overview KPI and a headline value.

### Costs (LLM & financial)

Aggregates and breakdowns.

| Intent | Endpoint |
|---|---|
| Total / weekly / monthly cost headline | **GET** `{dm_url}/api/analytics/financial/?metric={total_costs\|weekly_costs\|monthly_costs}&date_filter=all_time` |
| Cost per day chart | **GET** `{dm_url}/api/analytics/financial/?metric=total_costs&date_filter=30d` |
| Cost breakdown | **GET** `{dm_url}/api/analytics/financial/details/?group_by={provider\|llm_model\|username\|mentor\|platform\|action}&date_filter=30d` |
| Cost breakdown, paginated | append `&metrics=total_costs,sessions&page=1&limit=25&search=jane` |
| Invoice CSV/JSON | **GET** `{dm_url}/api/analytics/financial/invoice/?date_filter=30d` |

`financial/` requires `metric`. `financial/details/` requires `group_by`.

### Users & engagement

Registered and active users, plus time-of-day patterns.

| Intent | Endpoint |
|---|---|
| Registered users | **GET** `{dm_url}/api/analytics/users/?metric=registered_users&date_filter=30d` |
| Active users | **GET** `{dm_url}/api/analytics/users/?metric=active_users&date_filter=30d` |
| Currently active | **GET** `{dm_url}/api/analytics/users/?metric=currently_active&date_filter=today` |
| Time-of-day heatmap | **GET** `{dm_url}/api/analytics/time/?date_filter=30d` |
| User table (search + paginate) | **GET** `{dm_url}/api/analytics/users/details/?search=jane&page=1&limit=25&date_filter=30d` |

### Topics & conversations

| Intent | Endpoint |
|---|---|
| Topic overview | **GET** `{dm_url}/api/analytics/topics/?date_filter=30d` |
| Topic bar chart / details | **GET** `{dm_url}/api/analytics/topics/details/?search=&page=1&limit=25&date_filter=30d` |
| Conversation counts | **GET** `{dm_url}/api/analytics/conversations/?metric=conversations&date_filter=30d` |
| Ratings timeseries | **GET** `{dm_url}/api/analytics/ratings/?date_filter=30d` |

### Transcripts

| Intent | Endpoint |
|---|---|
| Transcript list | **GET** `{dm_url}/api/analytics/messages/?search={user_or_text}&sentiment={positive\|negative\|neutral}&topic={topic}&page=1&limit=20&date_filter=30d` |
| One transcript | **GET** `{dm_url}/api/analytics/messages/details/?session_id={id}` |

`messages/details/` requires `session_id`.

### Sessions & ratings

`sessions/` and `ratings/` both take `metric` and `date_filter` and return timeseries for line charts. See the schema for the exact response shape.

### Course analytics

Course-level analytics use `content/` with `metric=courses` (or `metric=course`).

| Intent | Endpoint |
|---|---|
| Course roster with enrolments / completions / ratings | **GET** `{dm_url}/api/analytics/content/?metric=courses&date_filter=30d` |
| Per-course drill-down | **GET** `{dm_url}/api/analytics/content/details/{course_id}/?metric=courses&time_metric={enrollments\|completions\|ratings\|time_spent}&date_filter=30d` |

`{course_id}` is the catalog identifier (the same `course_id` `/api/catalog/` returns). `content/details/` requires `metric`.

### Program analytics

Program-level analytics use the same family, with `metric=programs` (or `metric=program`).

| Intent | Endpoint |
|---|---|
| Program roster | **GET** `{dm_url}/api/analytics/content/?metric=programs&date_filter=30d` |
| Per-program drill-down | **GET** `{dm_url}/api/analytics/content/details/{program_id}/?metric=programs&time_metric={enrollments\|completions\|ratings\|time_spent}` |

### Pathway & skill analytics

Same shape as course/program. Use `metric=pathways` or `metric=skills` on `content/`, and `metric=pathway` / `metric=skill` on `content/details/`.

### Per-learner drill-down

One learner's learning data.

| Intent | Endpoint |
|---|---|
| Cross-platform learner summary | **GET** `{dm_url}/api/analytics/learners/?username={username}&date_filter=30d&page=1&limit=25` |
| Sortable, filterable learner roster | **GET** `{dm_url}/api/analytics/learners/list/?platform_key={platform_key}&search=&sort_by={username\|name\|last_activity\|total_points\|total_time_spent_seconds\|total_enrollments\|total_skills_count}&sort_order={asc\|desc}&page=1&limit=25` |
| Detailed learner profile | **GET** `{dm_url}/api/analytics/learner/details?username={username}&metrics={comma-separated}&include_edx_progress=true&course_id={id}&program_id={id}&pathway_id={id}&date_filter=30d` |

`learners/list/` requires `platform_key`. `learner/details` takes **no trailing slash** — Django URL registration. A learner may read their own record with the same endpoints; self-access needs no admin role.

### Time-on-platform

| Intent | Endpoint |
|---|---|
| Time spent per user | **GET** `{dm_url}/api/analytics/time-spent/user/` |

### Data Reports (list, poll, download)

Report reads. See `## Writes → Data Reports` below for report creation.

| Intent | Endpoint |
|---|---|
| List reports on the platform | **GET** `{dm_url}/api/reports/platforms/{platform_key}/?mentor_id={optional_pk}` — inlines the last-run status per report where one exists. |
| Poll report status | **GET** `{dm_url}/api/reports/platforms/{platform_key}/{report_name}?mentor_unique_id={optional_uuid}` — poll until `state == "completed"`; `url` then holds the download link. |
| Download a completed report | **GET** `{dm_url}/api/reports/platforms/{platform_key}/{task_id}/download?format={csv\|json}&columns={comma-separated}` — streams the finished output. `format` defaults to `csv`. `columns` reorders columns; leave empty for the default order. |

## Writes

### Push time-on-platform (client beacon)

- **POST** `{dm_url}/api/analytics/orgs/{org}/time/update/` (the wire segments are literal — `{org}` holds the platform key value).
  ```json
  {
    "time_spent_seconds": 120,
    "session_id": "…",
    "url": "…"
  }
  ```

### Data Reports (create)

- **POST** `{dm_url}/api/reports/platforms/{platform_key}/new` — kick off a report. **Confirm with the user first.**
  ```json
  {
    "report_name": "ai-mentor-chat-history",
    "mentor": "9e3b5c17-…",
    "usergroup_ids": [12, 34],
    "start_date": "2026-06-01",
    "end_date": "2026-06-30",
    "course_id": "course-v1:iblx+cs101+2026",
    "source": "",
    "query": ""
  }
  ```
  All fields are optional; supply what the target report expects. Mentor-scoped reports include `ai-mentor-chat-history`, `my-chat-history`, and `recommendation-history-report`. The response is a status object with `report_id`, `report_name`, `state`, `started_on`, `owner`, `expires`, and `poll_timeout`. See `## Reads → Data Reports` for poll and download.

## Example

Platform-wide messages/topics KPIs, then the same metric scoped to one agent:

```bash
curl -s "https://api.iblai.app/dm/api/analytics/topics/?date_filter=30d&platform_key=$IBLAI_ORG" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"

curl -s "https://api.iblai.app/dm/api/analytics/topics/?date_filter=30d&platform_key=$IBLAI_ORG&mentor_unique_id=$MENTOR" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

End-to-end chat-history Report — generate, poll, download:

```bash
# 1. Kick off
REPORT=$(curl -s -X POST "https://api.iblai.app/dm/api/reports/platforms/$IBLAI_ORG/new" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"report_name":"ai-mentor-chat-history","mentor":"'"$MENTOR"'","start_date":"2026-06-01","end_date":"2026-06-30"}')
NAME=$(echo "$REPORT" | jq -r '.data.status.report_name')

# 2. Poll
while true; do
  STATUS=$(curl -s "https://api.iblai.app/dm/api/reports/platforms/$IBLAI_ORG/$NAME?mentor_unique_id=$MENTOR" \
    -H "Authorization: Api-Token $IBLAI_API_KEY")
  STATE=$(echo "$STATUS" | jq -r '.data.state')
  [ "$STATE" = "completed" ] && break
  [ "$STATE" = "error" ] || [ "$STATE" = "cancelled" ] || [ "$STATE" = "expired" ] && { echo "failed: $STATE"; exit 1; }
  sleep 5
done

# 3. Download
TASK=$(echo "$STATUS" | jq -r '.data.report_id')
curl -s "https://api.iblai.app/dm/api/reports/platforms/$IBLAI_ORG/$TASK/download?format=csv" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -o chat-history.csv
```

## Notes

- `mentor_unique_id` alone toggles agent scope versus platform scope.
- `date_filter=custom` requires both `start_date` and `end_date` (`yyyy-MM-dd`).
- `learner/details` takes no trailing slash — Django's URL registration matches exactly.
- `learners/list/` requires `platform_key` in the query string.
- `financial/`, `financial/details/`, `content/details/`, and `messages/details/` each require one parameter (see the tables). The schema returns `400` if omitted.
- Report state `expired` means the download is gone; re-`POST` `/new` to regenerate.
- The download endpoint streams — expect a large body on chat-history reports.
- To discover agents, courses, programs, or users, see `/iblai-search`. For a user's own profile view of their analytics, use `/iblai-profile`.
