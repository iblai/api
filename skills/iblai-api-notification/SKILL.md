---
name: iblai-api-notification
description: Read and send ibl.ai platform notifications via the API — unread counts, the notifications list with filters, mark-as-read (all or specific), the two-step notification builder (preview then send/schedule, with source validation and recipient preview), per-user push-device (FCM) tokens, and org-admin email-template & SMTP management. Use when reading an organization's notifications, sending one, or configuring notification templates.
---

# iblai-api-notification

Read and send an organization's platform notifications via the API: the unread
count, the notifications list with channel and status filters, mark-as-read, and
the two-step notification builder (preview then send/schedule, with source
validation and recipient preview). Also manage per-user push-device (FCM) tokens
and, for org admins, the email templates and SMTP config that back notifications.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{platform_key}` = the org/platform key (usually `$IBLAI_ORG`).
- These are **platform-level** endpoints on the DM host; all paths below sit under
  `https://api.iblai.app/dm/api/notification/v1/` (written `…` in this doc).
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/users/{username}/notifications-count/?status=UNREAD` — unread count (also `channel`).
- **GET** `https://api.iblai.app/dm/api/notification/v1/orgs/{org}/users/{username}/notifications/` — notifications list (filters: `channel`, `status`, `start_date`, `end_date`, `exclude_channel`).
- **GET** `…/orgs/{org}/notification-builder/context/` — template/context variables available to the builder.
- **GET** `…/orgs/{org}/notification-builder/{build_id}/recipients/?page={n}&page_size={n}` — paged recipient preview for a built notification.

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
- **POST** `…/orgs/{org}/notification-builder/validate_source/` — validate recipient sources before building: body `{ "type": "string", "data": … }` → `{ valid_count, invalid_entries, sample_recipients }`.
- **PUT** `…/orgs/{org}/users/{username}/notifications/` — mark a specific notification read/unread: `{ "notification_id": "uuid", "status": "READ|UNREAD" }`.
- **POST** `…/orgs/{org}/users/{username}/register-fcm-token/` — register a push device: `{ "name": "string", "registration_id": "string" }`.
- **DELETE** `…/orgs/{org}/users/{username}/register-fcm-token/` — unregister a push device (same body).

## Template & SMTP admin (platform-scoped)

Org-admin management of email templates and SMTP, under
`…/platforms/{platform_key}/…`. `{type}` is a template key such as
`USER_NOTIF_COURSE_ENROLLMENT` or `USER_NOTIF_CREDENTIALS`.

- **POST** `…/platforms/{platform_key}/config/test-smtp/` — send a test email to verify SMTP: `{ smtp_host, smtp_port, smtp_username, smtp_password, use_tls, use_ssl, test_email, from_email }`.
- **PATCH** `…/platforms/{platform_key}/templates/{type}/` — edit a template: `{ email_subject, email_html_template, message_title, message }`.
- **POST** `…/platforms/{platform_key}/templates/{type}/reset/` — reset the template to its default.
- **POST** `…/platforms/{platform_key}/templates/{type}/test/` — send a test render: `{ context, course_name, credential_url }`.
- **PATCH** `…/platforms/{platform_key}/templates/{type}/toggle/` — enable/disable: `{ "is_enabled": bool }`.

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
