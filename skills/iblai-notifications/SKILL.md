---
name: iblai-notifications
description: Read and send ibl.ai platform notifications via the API — unread counts, the notifications list with filters, mark-as-read, and the two-step notification builder (preview then send/schedule). Use when reading an organization's notifications or sending one to recipients.
---

# iblai-notifications

Read and send an organization's platform notifications via the API: the unread
count, the notifications list with channel and status filters, mark-as-read, and
the two-step notification builder (preview then send/schedule). Use when reading
an organization's notifications or sending one to recipients.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- These are **platform-level** endpoints — the host is DM under
  `/api/notification/v1/…`.
- Not connected yet? Run **`/iblai-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/users/{username}/notifications-count/?status=UNREAD` — unread count (also `channel`).
- **GET** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/users/{username}/notifications/` — notifications list (filters: `channel`, `status`, `start_date`, `end_date`, `exclude_channel`).

## Writes

- **POST** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/mark-all-as-read` — mark all (or specific) notifications as read:
  ```json
  {
    "notification_ids": "uuid[] (omit/empty = mark ALL unread)"
  }
  ```
- **POST** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/notification-builder/preview/` — builder step 1, preview (returns `build_id`):
  ```json
  {
    "channels": "integer[] (required, e.g. [1])",
    "sources": "NotificationSource[] (required, from recipient emails)",
    "template_id": "uuid|null",
    "template_data": "object|null",
    "context": "object",
    "process_on": "ISO datetime|null (schedule)"
  }
  ```
- **POST** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/notification-builder/send/` — step 2, **send/schedule**:
  ```json
  {
    "build_id": "string (required)"
  }
  ```

## Example

Check the unread notification count for the current user:

```bash
curl -s \
  "https://api.iblai.app/dm/api/notification/v1/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/notifications-count/?status=UNREAD" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- Sending a notification is **outward-facing** — confirm the recipients with the
  user before the send step.
- The builder is two steps: `preview/` returns a `build_id`, then `send/` takes
  that `build_id` to actually send or schedule.
- `mark-all-as-read` with an empty/omitted `notification_ids` marks **all** unread
  notifications read; pass specific ids to mark just those.
