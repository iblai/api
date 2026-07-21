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

## Concepts

**`metadata` → `client_context` passthrough.** Every chat turn (WS or SSE) may carry a
`metadata` object of arbitrary key/values (`BaseConsumerPayload.metadata`). The runner
folds it into the prompt the agent sees, so the agent can tailor its reply, and the
consumer persists it on the session as `Session.metadata["client_context"]`. It is
**session-level**: each turn's `metadata` overwrites the session's `client_context`, so
it sticks across turns until you send new keys. Use it to tell one deployed agent
*where/why* a message arrives (product, plan tier, page, region) without editing its
prompt. It then echoes back on every read below. The sibling field `page_content` is
also appended to the prompt, but — unlike `metadata` — is stripped before the message is
saved.

## Reads

- **GET** `…/dm/api/ai-mentor/orgs/{org}/users/{user}/sessions/` — list the user's
  chat sessions.
- **GET** `…/orgs/{org}/users/{user}/sessions/{session_id}/` — the session's paginated
  chat messages (`MessageView`); the response also carries `client_context`, read from
  the session's `metadata["client_context"]`.
- **GET** `…/orgs/{org}/users/{user}/sessions/{session_id}/tasks/{task_id}/` — the
  chat-history export (`DownloadableChatHistory`); every item carries a `client_context`
  field. Add `?to_csv=true` for a CSV whose columns are exactly
  `type,content,timestamp,client_context`. Kicking off (POST) and polling that export
  task is owned by **`/iblai-api-agent-history`**.
- **Analytics echo:** the same value comes back at `summary.client_context` from
  **GET** `…/dm/api/analytics/messages/details/?platform_key={org}&session_id={session_id}`
  — documented under **`/iblai-api-analytics`**.

## Writes

- **POST** `https://asgi.data.iblai.app/api/agent/chat/?platform_key={org}&session_id={session_id}`
  — send a chat turn; response is Server-Sent Events. Body (`BaseConsumerPayload`):
  ```json
  {
    "session_id": "…",
    "prompt": "Hello",
    "flow": { "name": "<agent unique_id>", "tenant": "<org key>" },
    "page_content": "optional text appended to the prompt, stripped before saving",
    "metadata": { "any": "client context keys" }
  }
  ```
  `session_id` (a UUID4) and `flow` are **required**; `flow.name` selects the agent (its
  `unique_id`, or a slug/name) and `flow.tenant` is the org key. `prompt` and
  `page_content` default to empty, `metadata` to null. `metadata` is passthrough — stored
  on the session as `client_context` (see Concepts) and echoed in the reads above.
  The **same payload** works over WebSocket at `wss://asgi.data.iblai.app/ws/chat/`.
- **POST** `…/dm/api/ai-mentor/orgs/{org}/users/{user}/sessions/` — create/retrieve a
  session (`ChatSessionView`; the body's `mentor` field picks the agent). Or let the
  first chat turn create one by passing a new `session_id`.

## Example

```bash
# Stream a chat turn with attached client context (SSE). MENTOR = the agent's unique_id.
curl -N -X POST \
  "https://asgi.data.iblai.app/api/agent/chat/?platform_key=$IBLAI_ORG&session_id=$SESSION" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"session_id":"'"$SESSION"'","prompt":"Summarize my notes","flow":{"name":"'"$MENTOR"'","tenant":"'"$IBLAI_ORG"'"},"metadata":{"source":"docs","tab":"notes"}}'

# Read the session's messages back (client_context is the metadata you sent)
curl "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/sessions/$SESSION/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"

# Download the history export as CSV (client_context is a column)
curl "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/sessions/$SESSION/tasks/$TASK/?to_csv=true" \
  -H "Authorization: Api-Token $IBLAI_API_KEY"
```

## Notes

- **Streaming runs on ASGI** — the chat turn (`/api/agent/chat/`, `/ws/chat/`) is on
  `asgi.data.iblai.app`; session reads are ordinary REST on the `api.iblai.app/dm`
  gateway. `session_id` and `flow` are required on every turn, and `flow.name` must
  resolve to a deployed agent (its `unique_id`, slug, or name).
- For **OpenAI-format** inference against a provider/model (no agent RAG/memory), use
  `/iblai-api-inference`; to chat via an **MCP server** instead of raw SSE, use
  `/iblai-api-agent-chat`; the history-export task (kick off + poll) is
  `/iblai-api-agent-history`; the `summary.client_context` analytics read is
  `/iblai-api-analytics`.

## Schema

**`metadata` / `client_context`** — an arbitrary JSON object (`dict[str, Any] | null`,
`BaseConsumerPayload.metadata`). No fixed keys; use whatever your app needs, e.g.
`product`, `planTier`, `userRole`, `region`. Sent as `metadata` on a chat turn, it is
persisted at `Session.metadata["client_context"]` and read back as `client_context` in
the session-messages response, the history export (a `client_context` field per item, or
CSV column via `?to_csv=true`), and analytics `summary.client_context`.

## Reference material

- [`references/metadata-passthrough.md`](references/metadata-passthrough.md) — the
  `metadata` pass-through companion: the `<CONTEXT METADATA>` prompt-injection format,
  per-transport wire notes (SSE/WebSocket + the embedded-iframe `postMessage` channel),
  session caching (send-once, replace-not-merge, ~2h TTL), the
  one-agent-many-contexts pattern, and the storage/pipeline map (session
  `client_context` vs. the per-message snapshot).
