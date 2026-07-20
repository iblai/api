---
name: iblai-api-agent-session
description: Talk to a deployed ibl.ai agent directly over REST/SSE (or WebSocket) and manage its chat sessions — POST a prompt to the agent chat endpoint, attach arbitrary metadata (surfaced later as client_context), and list/read sessions and per-task history exports. The direct-transport counterpart to iblai-api-agent-chat's MCP wiring; use when you want raw streamed chat + session records rather than an MCP server.
---

# iblai-api-agent-session

Drive a deployed agent's **chat transport directly** and read its **sessions**.
Where `/iblai-api-agent-chat` wires a hosted MCP server for conversation, this
skill is the raw REST/SSE (and WebSocket) surface: POST a prompt, stream the
reply, attach `metadata` that resurfaces as `client_context`, and list/inspect
the resulting session records. Get `IBLAI_ORG`/`IBLAI_USERNAME`/`IBLAI_API_KEY`
from `/iblai-api-login`.

## Auth & conventions

- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{user}` = `$IBLAI_USERNAME`.
- **Two hosts — chat is streaming/ASGI:**
  - Chat turn (SSE / WebSocket) → `https://asgi.data.iblai.app`
  - Session reads/writes → `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{user}/v1` … i.e. `…/orgs/{org}/users/{user}/sessions/…`
- Not connected yet? Run **`/iblai-api-login`** first.

## Reads

- **GET** `…/dm/api/ai-mentor/orgs/{org}/users/{user}/sessions/` — list the user's
  chat sessions.
- **GET** `…/orgs/{org}/users/{user}/sessions/{session_id}/` — session detail;
  includes `client_context` (the `metadata` sent on the chat turn).
- **GET** `…/orgs/{org}/users/{user}/sessions/{session_id}/tasks/{task_id}/` —
  a session task / chat-history export (CSV) — carries a `client_context` column.

## Writes

- **POST** `https://asgi.data.iblai.app/api/agent/chat/?platform_key={org}&session_id={session_id}`
  — send a chat turn; response is Server-Sent Events. Body:
  ```json
  {
    "session_id": "…",
    "prompt": "Hello",
    "metadata": { "any": "client context keys" }
  }
  ```
  `metadata` is passthrough — it is stored on the session as `client_context` and
  echoed in analytics (`/iblai-api-analytics` → `messages/details` `summary.client_context`).
  A WebSocket transport is also available at `wss://asgi.data.iblai.app/ws/chat/`.
- **POST** `…/dm/api/ai-mentor/orgs/{org}/users/{user}/sessions/` — create a
  session (or let the first chat turn create one by passing a new `session_id`).

## Example

```bash
# Stream a chat turn with attached client context (SSE)
curl -N -X POST \
  "https://asgi.data.iblai.app/api/agent/chat/?platform_key=$IBLAI_ORG&session_id=$SESSION" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"session_id":"'"$SESSION"'","prompt":"Summarize my notes","metadata":{"source":"docs","tab":"notes"}}'

# Read the session back (client_context is the metadata you sent)
curl "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/sessions/$SESSION/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- **Streaming is ASGI-only** — the chat turn (`/api/agent/chat/`, `/ws/chat/`) runs
  on `asgi.data.iblai.app`; session reads are ordinary REST on the `api.iblai.app/dm`
  gateway.
- **`metadata` → `client_context`** is arbitrary key/values you attach per turn; it
  round-trips into session detail, the task export column, and analytics
  `messages/details` — use it to tag where/why a message came from.
- For **OpenAI-format** inference against a provider/model (no agent RAG/memory),
  use `/iblai-api-inference`. To chat via an **MCP server** instead of raw SSE, use
  `/iblai-api-agent-chat`.
