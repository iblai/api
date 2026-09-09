---
name: iblai-api-exim
description: Export and import ibl.ai data via the platform API — start and poll whole-org migration runs (export a bundle, download the .zip, upload it to another deployment to import), plus course-level export eligibility, Studio target environments, and the export/import task queue. Use when moving an organization between deployments, pulling your data out as a portable bundle, or inspecting course export/import tasks.
---

# iblai-api-exim

Move ibl.ai data in and out of a deployment over REST. Two independent lanes
live under `/api/exim/manage/`:

- **Platform migrations** — whole-organization export/import. Start a run, poll
  it, download the resulting bundle `.zip`, and upload that same zip to another
  deployment to import it. This is the "take my data with me" path.
- **Course export/import** — per-course publishing to a Studio (CMS) target
  environment. You declare which courses are eligible and which environments
  they can go to; the platform queues an `EximTask` per publish and this API
  surfaces those tasks for inspection and cancellation.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Prefix:** every endpoint below is a Data Manager route — call it as
  `https://api.iblai.app/dm/api/exim/manage/…`.
- **Permission:** every endpoint requires **DM admin** (`IsDMAdmin`). A
  non-admin token gets `403`, not an empty list.
- **Org scope:** migration runs identify orgs by **org key** on the wire as
  `source_key` (export) and `target_key` (import) — the same value as
  `$IBLAI_ORG`.
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.
- Destructive or outward-facing calls below (delete an environment or
  eligibility row, cancel a run or task, start an import that writes into a
  target org) are marked — **confirm with the user first.**

## Concepts

- **A run is async.** `POST …/platform-migrations/` validates, creates the row,
  returns `201` with a `run_id`, and dispatches a Celery worker. Everything
  after that is polling `GET …/platform-migrations/{run_id}/`.
- **`run_id` is the lookup key**, not the surrogate PK — detail, cancel, and
  download all take the `run_id` UUID.
- **`status`** is one of `pending` · `in_progress` · `completed` · `failed` ·
  `canceled`. **`phase`** is the human-readable step inside a run, and
  **`report`** carries per-phase summaries, component selection, id-map totals,
  and warnings once the run finishes.
- **Components** select how much of the org travels. Mandatory sets (`core`,
  `mentors`, `finalize`) always run; the optional sets are opt-in. Omitting
  `selected_components` entirely means **full fidelity** (everything). See
  [references/platform-migration.md](references/platform-migration.md).
- **Feature-gated.** Starting a run (`POST …/platform-migrations/` or
  `…/import-upload/`) returns `503 {"detail": "Platform migration is disabled
  on this deployment."}` where the feature is off. Reads still work.
- **Cancel only catches a pending run.** Once the worker flips it to
  `in_progress`, cancel returns `409` and the run is not interrupted mid-phase.

## Reads

### Platform migrations

- **GET** `https://api.iblai.app/dm/api/exim/manage/platform-migrations/` — list runs. Filters: `?direction=export|import`, `?status=pending|in_progress|completed|failed|canceled`, `?source_key=`, `?target_key=`. Ordering: `?ordering=` over `created_at`, `started_at`, `finished_at` (default `-created_at`).
- **GET** `…/platform-migrations/{run_id}/` — one run: `run_id`, `direction`, `source_key`, `target_key`, `status`, `phase`, `cursor`, `selected_components`, `report`, `error_message`, `download_url`, `bundle_path`, `bundle_storage_key`, `admin_user`, `email`, `created_at`, `updated_at`, `started_at`, `finished_at`. **This is the poll endpoint.**
- **GET** `…/platform-migrations/{run_id}/download/` — stream the bundle as a single `.zip`. Only a **completed export** is downloadable: an import returns `400`, an unfinished run `409`, and a run whose bundle is gone `404`. On an S3 deployment this `302`-redirects to a presigned URL — follow redirects (`curl -L`).

> `download_url` on the run object is the same thing precomputed: a presigned S3
> URL when the bundle lives in S3, otherwise the local download endpoint, and
> `null` when there is nothing to pull.

### Course export tasks

- **GET** `…/tasks/` — list export/import tasks. Filters: `?status=`, `?target_environment={id}`, `?course__course_id=`. Ordering over `created_at`, `published_at`, `started_at`, `finished_at` (default `-created_at`).
- **GET** `…/tasks/{id}/` — one task: `id`, `course`, `course_id`, `target_environment`, `target_environment_name`, `status`, `published_at`, `export_download_url`, `import_filename`, `error_message`, plus timestamps. **Read-only** — tasks are created by publish signals, never by this API.

### Eligibility & environments

- **GET** `…/eligibility/` — which courses may be exported. Filters: `?is_enabled=true|false`, `?course__course_id=`; `?search=` matches `course__course_id`.
- **GET** `…/eligibility/{id}/` — one eligibility record: `id`, `course`, `course_id`, `is_enabled`, `target_environments`, `created_at`, `updated_at`.
- **GET** `…/environments/` — Studio targets. Filters: `?is_enabled=true|false`; `?search=` matches `name` and `studio_base_url`.
- **GET** `…/environments/{id}/` — one environment: `id`, `name`, `studio_base_url`, `is_enabled`, `oauth_credentials`. **`token` is write-only** and never comes back in a response.

## Writes

### Platform migrations

- **POST** `…/platform-migrations/` — start a run. Returns `201` with the full run object (poll `run_id` from there).
  ```json
  {
    "direction": "export",
    "source_key": "$IBLAI_ORG",
    "selected_components": ["conversations", "memory"],
    "email": "admin@example.com"
  }
  ```
  - `direction` **required** — `export` or `import`.
  - `source_key` **required for `export`**; `target_key` **required for `import`** (validation error names the missing one).
  - `selected_components` optional — omit for full fidelity. Unknown keys are rejected with the valid optional key list in the error.
  - `bundle_path` optional — backfilled server-side from a deterministic root; pin it only to replay a specific bundle.
  - `admin_user` / `email` optional — attribution and completion notice.
  - **An import writes into the target org. Confirm with the user first.**
- **POST** `…/platform-migrations/import-upload/` — upload a bundle `.zip` and start the import from it, in one call. `multipart/form-data`: `bundle_file` (the zip, **required**), `target_key` (**required**), optional repeatable `selected_components`, plus `admin_user`, `email`. `source_key` is read from the bundle itself. Rejects a zip with no `manifest.json` (`400` — "not a migration bundle"). **Confirm with the user first.**
- **POST** `…/platform-migrations/{run_id}/cancel/` — cancel a **pending** run. Returns the updated run, or `409` if the worker already started it. **Confirm with the user first.**

### Course export tasks

- **POST** `…/tasks/{id}/cancel/` — cancel a **pending** task so the worker skips it. `409` once it is `in_progress`. **Confirm with the user first.**

### Eligibility & environments

- **POST** `…/eligibility/` — make a course exportable: `{"course": <id>, "is_enabled": true, "target_environments": [<env id>, …]}`. `course_id` is read-only (derived from `course`).
- **PUT / PATCH** `…/eligibility/{id}/` — update; `PATCH` for a single field.
- **DELETE** `…/eligibility/{id}/` — remove the eligibility record. **Confirm with the user first.**
- **POST** `…/environments/` — register a Studio target:
  ```json
  {
    "name": "Production Studio",
    "studio_base_url": "https://studio.learn.example.com",
    "is_enabled": true,
    "token": "<bearer token>"
  }
  ```
  `name` and `studio_base_url` are required, and you must supply **either**
  `oauth_credentials` (an OAuth credentials id) **or** a bearer `token` —
  neither present is a validation error.
- **PUT / PATCH** `…/environments/{id}/` — update. Re-sending `token` replaces it; it is never readable back.
- **DELETE** `…/environments/{id}/` — remove the target environment. **Confirm with the user first.**

## Example

Export the connected org, poll until it finishes, then pull the bundle:

```bash
# 1. start the export — capture the run_id
RUN_ID=$(curl -s -X POST \
  "https://api.iblai.app/dm/api/exim/manage/platform-migrations/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"direction\":\"export\",\"source_key\":\"$IBLAI_ORG\"}" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["run_id"])')

# 2. poll until status leaves pending/in_progress
while :; do
  STATUS=$(curl -s \
    "https://api.iblai.app/dm/api/exim/manage/platform-migrations/$RUN_ID/" \
    -H "Authorization: Api-Token $IBLAI_API_KEY" \
    | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["status"], d["phase"])')
  echo "$STATUS"
  case "$STATUS" in pending*|in_progress*) sleep 10 ;; *) break ;; esac
done

# 3. download the bundle (-L follows the S3 presigned redirect)
curl -sL -o bundle.zip \
  "https://api.iblai.app/dm/api/exim/manage/platform-migrations/$RUN_ID/download/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

Then, on the destination deployment, import that same zip:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/exim/manage/platform-migrations/import-upload/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -F "bundle_file=@bundle.zip" \
  -F "target_key=$IBLAI_ORG"
```

## Notes

- **Poll, don't assume.** `POST` returns `201` the moment the row commits — the
  work has not started. A run is only finished when `status` is `completed`,
  `failed`, or `canceled`; read `report` on success and `error_message` on
  failure.
- **The two lanes are unrelated.** `platform-migrations` moves a whole
  organization between deployments. `tasks` / `eligibility` / `environments`
  publish individual courses to a Studio CMS. Nothing in one lane affects the
  other.
- **Tasks are never created through this API** — they come from the course
  publish signal and the admin sync action. `cancel` is the only write.
- **`token` on an environment is write-only.** Reading an environment back to
  "check" its token will always show it absent; re-send the field to rotate it.
- **Component selection is asymmetric.** Mandatory sets run whether or not you
  name them, so `selected_components: ["memory"]` still exports `core`,
  `mentors`, and `finalize` — it just drops the other optional sets.
- **Large bundles.** The download streams a zip built on demand; on multi-node
  or S3 deployments it redirects to S3 instead, so always follow redirects and
  write to a file rather than buffering in memory.

## Reference material

- [`references/platform-migration.md`](references/platform-migration.md) — the
  run lifecycle end to end, the component registry (which data sets are
  mandatory vs optional, and what each carries), status/phase semantics, and
  the deployment-to-deployment handoff.
