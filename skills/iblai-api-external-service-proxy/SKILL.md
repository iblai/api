---
name: iblai-api-external-service-proxy
description: Call third-party AI services (ElevenLabs text-to-speech & voices, HeyGen avatar video) through ibl.ai's External Service Proxy. Discover the available services and their endpoints, then POST a request envelope (body/query/path_params/files) to invoke one — the provider API key is stored server-side per org, so clients never hold it. Use for TTS audio, voice/avatar/template listing, and avatar video generation with status polling.
---

# iblai-api-external-service-proxy

A service-agnostic **gateway** for calling third-party AI providers (ElevenLabs,
HeyGen, …) *through* ibl.ai instead of hitting them directly. One request shape
fronts every provider; the provider's API key is stored server-side per org and
injected upstream, so the client never holds it. Work in two phases: **discover**
a service's endpoints, then **invoke** one. Configure the provider keys with
`/iblai-api-integration`; get `IBLAI_ORG`/`IBLAI_API_KEY` from `/iblai-api-login`.

## Auth & conventions

- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Base:** `https://api.iblai.app/dm/api/ai-proxy/orgs/{org}` — `{org}` = `$IBLAI_ORG`.
  (App segment is `ai-proxy`; backend routes are bare `/api/...`, the gateway
  prepends `/dm`.)
- **Invoke envelope** — every invoke is `POST .../services/{service}/{action}/`
  with a JSON envelope; all fields optional, forwarded to the upstream provider:
  ```json
  {
    "body": {},         // JSON body sent to the provider (its own schema)
    "query": {},        // upstream query-string params
    "headers": {},      // extra upstream headers
    "path_params": {},  // fills {placeholders} in the endpoint's path_template
    "files": {}         // multipart uploads
  }
  ```
- **Response mode** is per-endpoint (from discovery): `json` (parse JSON),
  `binary` (audio/video/image blob — write to a file, don't JSON-parse),
  `passthrough` (check `Content-Type`), or `stream` (SSE/chunked).
- Provider **credentials are configured out-of-band** (admin / `/iblai-api-integration`),
  not through this API. Resolution follows the service's `credential_policy`
  (tenant vs platform key, with optional fallback).

## Reads (discover)

- **GET** `…/orgs/{org}/services/` — list available services (each `slug`,
  `service_type`, whether enabled).
- **GET** `…/orgs/{org}/services/{service}/` — service detail: its endpoints
  (`slug`, `http_method`, `path_template`, `request_mode`, `response_mode`,
  `supports_streaming`) plus `credential_policy` / `credential_schema`.

## Writes (invoke)

- **POST** `…/orgs/{org}/services/{service}/{action}/` — invoke an endpoint with
  the envelope above. `{service}` = provider slug, `{action}` = endpoint slug.

**ElevenLabs** (`service: elevenlabs`, TTS):

| action | upstream | envelope | returns |
|--------|----------|----------|---------|
| `list-voices` | `GET /v1/voices` | `{}` | `{voices:[{voice_id,name,category,labels}]}` |
| `list-models` | `GET /v1/models` | `{}` | `[{model_id,name,description}]` |
| `tts` | `POST /v1/text-to-speech/{voice_id}` | `path_params.voice_id` + `body.{text,model_id,voice_settings?}` | **binary** `audio/mpeg` |

**HeyGen** (`service: heygen`, avatar video — async: generate, then poll):

| action | envelope | returns |
|--------|----------|---------|
| `list-templates` / `list-avatars` / `list-voices` | `{}` | `data.{templates|avatars|voices}[]` |
| `generate-video` | `body.{test?,video_inputs:[{character,voice}],dimension}` | `data.video_id` |
| `generate-template-video` | `path_params.template_id` + `body.{test?,caption?,variables}` | `data.video_id` |
| `video-status` | `query.video_id` | `data.status` = `processing` \| `completed`(+`video_url`) \| `failed`(+`error`) |

## Examples

```bash
# discover
curl "https://api.iblai.app/dm/api/ai-proxy/orgs/$IBLAI_ORG/services/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
curl "https://api.iblai.app/dm/api/ai-proxy/orgs/$IBLAI_ORG/services/elevenlabs/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"

# ElevenLabs TTS -> MP3 (binary; write to file)
curl -X POST "https://api.iblai.app/dm/api/ai-proxy/orgs/$IBLAI_ORG/services/elevenlabs/tts/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"path_params":{"voice_id":"21m00Tcm4TlvDq8ikWAM"},"body":{"text":"Hello, this is a test.","model_id":"eleven_multilingual_v2","voice_settings":{"stability":0.5,"similarity_boost":0.5}}}' \
  --output speech.mp3

# HeyGen: generate avatar video (test:true = no credits) -> video_id, then poll
curl -X POST "https://api.iblai.app/dm/api/ai-proxy/orgs/$IBLAI_ORG/services/heygen/generate-video/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"body":{"test":true,"video_inputs":[{"character":{"type":"avatar","avatar_id":"Abigail_expressive_2024112501","avatar_style":"normal"},"voice":{"type":"text","input_text":"Hello.","voice_id":"f38a635bee7a4d1f9b0a654a31d050d2"}}],"dimension":{"width":1280,"height":720}}}'

curl -X POST "https://api.iblai.app/dm/api/ai-proxy/orgs/$IBLAI_ORG/services/heygen/video-status/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" -H "Content-Type: application/json" \
  -d '{"query":{"video_id":"video_123abc"}}'
```

## Notes

- **`body` is forwarded verbatim** to the provider — its schema is the provider's
  own (ElevenLabs / HeyGen API docs), the proxy doesn't reshape it. A
  `path_template` placeholder (e.g. `{voice_id}`) means that `path_params` key is
  required; omit it → `400 Missing required path parameter`.
- **Binary responses** (`tts`) are raw bytes — save to a file, don't parse as JSON.
- **HeyGen is async:** `generate-*` returns a `video_id`; poll `video-status`
  until `completed` for the `video_url`. Use `test:true` to avoid spending credits.
- **Errors:** body is keyed by `detail` *or* `error` (read `detail || error`).
  `404 …credentials found…` = the provider key isn't configured for this org
  (set it via `/iblai-api-integration`); `502` = upstream provider failure/quota;
  `429`/`502` → retry with exponential backoff.
- The proxy's own docs also show `Authorization: Api-Key <key>` / `Token <token>`
  for the platform token — the house form here is `Api-Token $IBLAI_API_KEY`; all
  are the platform token, **not** the provider key (that lives server-side).
