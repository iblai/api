# Analytics schema — priority, fetch, refresh, drift check

The OpenAPI schema is the contract for every endpoint the `iblai-analytics` skill references. Parameter names, enums, and response shapes come from the schema — not from prose in `SKILL.md`.

## Priority

1. **Live schema (authoritative):** `https://api.iblai.app/dm/api/docs/schema/?format=json`.
2. **Local snapshot (offline bootstrap):** `references/analytics-schema.json`. Filtered to `/api/analytics/*` and `/api/reports/*`. Refresh on demand (see below). The live schema wins on any disagreement.

## Fetch the live schema

```bash
curl -s -H "Accept: application/json" \
     "https://api.iblai.app/dm/api/docs/schema/?format=json" \
     > /tmp/iblai-schema.json
```

Filter to analytics + reports paths:

```bash
jq '.paths | with_entries(select(.key | test("^/api/(analytics|reports)/")))' \
   /tmp/iblai-schema.json \
   > /tmp/iblai-analytics-schema.json
```

## Refresh the local snapshot

Run from the repo root to regenerate `skills/iblai-analytics/references/analytics-schema.json`:

```bash
curl -s -H "Accept: application/json" \
     "https://api.iblai.app/dm/api/docs/schema/?format=json" \
  | jq -c '{
      openapi: .openapi,
      info: {
        title: .info.title,
        version: .info.version,
        note: "Snapshot filtered to /api/analytics/* and /api/reports/* paths. Component $refs point back to the live schema at https://api.iblai.app/dm/api/docs/schema/?format=json (authoritative)."
      },
      paths: (.paths | with_entries(select(.key | test("^/api/(analytics|reports)/"))))
    }' \
  > skills/iblai-analytics/references/analytics-schema.json
```

The snapshot omits `components.schemas` to stay small. Any `$ref` in the snapshot points to the live schema — resolve it there.

## Inspect one endpoint

Query parameters:

```bash
jq '.["/api/analytics/financial/"].get.parameters' \
   skills/iblai-analytics/references/analytics-schema.json
```

Response shape (200):

```bash
jq '.["/api/analytics/financial/"].get.responses["200"]' \
   skills/iblai-analytics/references/analytics-schema.json
```

Required request body for POST (Reports):

```bash
jq '.["/api/reports/platforms/{key}/new"].post.requestBody' \
   skills/iblai-analytics/references/analytics-schema.json
```

## List every documented path

```bash
jq -r '.paths | keys[]' \
   skills/iblai-analytics/references/analytics-schema.json \
  | sort
```

## Drift check before shipping

Every URL cited in `SKILL.md` must appear in the schema. The check has to account for two spelling differences between the skill's prose and the schema's paths:

1. The skill uses the `{dm_url}` anchor (= `https://api.iblai.app/dm`) plus `/api/…` — rewrite `{dm_url}/api` to `/api` before comparing. Legacy `…` shorthand also supported for older skills.
2. Some placeholders differ: the skill says `{platform_key}` where the schema uses `{key}`; the skill uses domain nouns like `{course_id}` and `{program_id}` where the schema uses the generic `{content_id}`. Normalize before comparing.

```bash
grep -oE '(\{dm_url\}|…|\.\.\.|https://api\.iblai\.app/dm)/api/(analytics|reports)/[a-zA-Z0-9/_<>{}-]*' \
     skills/iblai-analytics/SKILL.md \
  | sed -E 's#^\{dm_url\}#/dm#; s#^…#/api#; s#^\.\.\.#/api#; s#^https://api\.iblai\.app/dm/api#/api#; s#^/dm/api#/api#; s/\{platform_key\}/{key}/g; s/\{course_id\}/{content_id}/g; s/\{program_id\}/{content_id}/g' \
  | sort -u \
  | while read -r url; do
      # skip incomplete stubs like /api/analytics/ on its own
      case "$url" in "/api/analytics/"|"/api/reports/platforms/") continue ;; esac
      jq -e --arg u "$url" '.paths | has($u)' \
         skills/iblai-analytics/references/analytics-schema.json > /dev/null \
        || echo "MISSING: $url"
    done
```

Any `MISSING:` line is a bug in the skill, not the schema. Fix `SKILL.md`; do not work around a schema mismatch client-side.

Also compare the local snapshot to the live schema — a `MISSING:` on the snapshot side means the snapshot is stale. Refresh it.

## When the schema disagrees with the skill

The schema wins. Update `SKILL.md` to match. If the change is behavioural (a new required parameter, a renamed enum value, a removed endpoint), also update the `## Notes` section so callers know.
