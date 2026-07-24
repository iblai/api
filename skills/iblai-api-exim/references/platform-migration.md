# Platform migration — lifecycle, components, and the deployment handoff

A **platform migration** moves one organization's data out of a deployment as a
portable bundle, and back into another deployment. It is the API behind "you own
your data" — a run produces a `.zip` you can hold, inspect, and replay somewhere
else. This is separate from the course export/import lane (`tasks/`,
`eligibility/`, `environments/`), which publishes individual courses to a Studio
CMS and shares nothing with migrations but the URL prefix.

## The run lifecycle

A run is a row plus a background worker. Nothing about it is synchronous.

1. **Start.** `POST …/platform-migrations/` validates the request, saves the row,
   and — only once that row commits — dispatches the Celery task. You get `201`
   with the full run object; `run_id` (a UUID) is the handle for everything else.
2. **Poll.** `GET …/platform-migrations/{run_id}/` returns `status` and `phase`.
   `status` moves `pending` → `in_progress` → one of `completed` / `failed` /
   `canceled`. `phase` names the step inside the run; `cursor` tracks progress
   within a phase.
3. **Finish.** On success `report` holds per-phase summaries, the resolved
   component selection, id-map totals, and warnings. On failure read
   `error_message`.
4. **Collect.** For a completed **export**, `download_url` is populated and
   `…/{run_id}/download/` streams the bundle.

Lookups are by `run_id`, not by the surrogate primary key — the PK is never
exposed.

### Cancellation is a race you usually lose

`POST …/{run_id}/cancel/` performs a conditional update that only matches a row
still in `pending`. If the worker has already picked the run up, zero rows match
and you get `409` with the current status. A run that is executing is **not**
interrupted mid-phase — there is no way to stop it once it starts. The same
mechanic applies to `POST …/tasks/{id}/cancel/`.

### Feature gating

Both start paths (`POST …/platform-migrations/` and `…/import-upload/`) check a
deployment feature flag and return
`503 {"detail": "Platform migration is disabled on this deployment."}` when it is
off. Reads are never gated — you can always list and inspect existing runs.

## The component registry

`selected_components` chooses how much of the org travels. Components run in a
fixed, FK-safe order; the selection cannot reorder them.

| Key | Carries | Mandatory? | Phase |
|---|---|---|---|
| `core` | Platform + users + links | **yes** | 2 |
| `mentors` | Agents, plus lookups, templates, settings, prompts, and M2M rows | **yes** | 3 |
| `conversations` | Topics, sessions, messages, feedback, pins | no | 4 |
| `memory` | Memory records and embeddings | no | 5 |
| `training_data` | Training data, for re-training on the far side | no | 5b |
| `blobs` | Blob bytes out of S3 | no | 6 |
| `finalize` | Redaction, FK verification, id-map, report | **yes** | 7 |

Rules that follow from that table:

- **Mandatory sets always run.** Naming `["memory"]` does not mean "only memory"
  — it means core + mentors + memory + finalize, and drops `conversations`,
  `training_data`, and `blobs`.
- **Omitting the field entirely means full fidelity** — every component, nothing
  dropped. That is the right default unless you have a reason to trim.
- **Unknown keys are rejected** at validation time, and the error lists the valid
  optional keys.
- Phase 1 (discover / preflight) is read-only and always runs, so it is not a
  selectable component.

Trimming is mostly about size and time: `blobs` and `memory` dominate a large
org's bundle, and `conversations` grows without bound on a busy deployment.
Trimming is also lossy in the obvious way — what you drop does not exist on the
far side.

## Moving a bundle between deployments

The two halves of a migration run on two different deployments and are joined by
a file you carry.

**On the source deployment**

1. `POST …/platform-migrations/` with `direction: "export"` and
   `source_key: <org key>`.
2. Poll until `status` is `completed`.
3. `GET …/platform-migrations/{run_id}/download/` to pull the `.zip`.

**On the destination deployment**

4. `POST …/platform-migrations/import-upload/` as `multipart/form-data` with
   `bundle_file` (the zip) and `target_key` (the destination org key).

The upload path validates the archive before it accepts it: the zip must contain
a `manifest.json`, or it is rejected with `400` and
`"no manifest.json in the zip — not a migration bundle."` The **`source_key` is
read out of the bundle**, not from your request — you only say where it is going.

If you would rather not push bytes through the API, `POST
…/platform-migrations/` with `direction: "import"` and a `bundle_path` that the
destination can already reach does the same thing from a path on disk. The
upload endpoint exists so a caller with nothing but API access can still complete
the handoff.

### Where the bundle actually lives

- **Single-node deployment.** The bundle sits on that node's disk at
  `bundle_path`, and `download` zips and streams it.
- **S3 / multi-node deployment.** The bundle is written to object storage under
  `bundle_storage_key`, and both `download_url` and the `download` endpoint hand
  back a **presigned S3 URL** instead. `download` answers with a `302`, so any
  client must follow redirects. The presigned URL needs no platform auth, which
  is what makes it usable from a node that never held the file.

`download_url` is `null` — and `download` refuses — in every case where there is
nothing to fetch: the run is an import (`400`), it has not completed (`409`), or
the bundle is no longer on disk (`404`).

## Reading a finished run

- `report` — the authoritative record of what happened: per-phase summaries,
  which components were resolved as active, id-map totals, and any warnings the
  runner raised. A run can complete *with* warnings; check them.
- `error_message` — populated on `failed`.
- `admin_user` / `email` — attribution captured at start; `email` is also where
  the completion notice goes.
- `started_at` / `finished_at` — null until the worker reaches each point, which
  is why a `pending` run has neither.
