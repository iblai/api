# Chat metadata pass-through

Explanatory companion to `/iblai-api-agent-session`. The endpoints, fields, and the
`metadata` → `client_context` model are authoritative in `SKILL.md` (`## Concepts`,
`## Reads`, `## Schema`); this file only adds what the endpoint reference doesn't — the
prompt-injection format, the per-transport wire notes (incl. the iframe channel), and
the caching semantics. Auth is unchanged: `Authorization: Api-Token $IBLAI_API_KEY`,
`{org}` = `$IBLAI_ORG`, `{user}` = `$IBLAI_USERNAME`.

## The idea

`metadata` is free-form JSON (no schema) you attach to a chat turn to tell one deployed
agent *where/why* a message arrives — product, plan tier, page, region — without editing
its prompt or making the user restate context. Optional: omit it and the agent behaves as
before. It is persisted per session as `client_context` and surfaced on every read (the
session read, the history export, and analytics `summary.client_context` — see
`SKILL.md ## Reads`, don't re-fetch from here).

The keys are your app's own — support: `product`/`planTier`/`region`; training:
`department`/`courseId`/`employeeLevel`; commerce: `category`/`brand`/`priceRange` — no
schema is enforced.

## What the agent sees

The runner appends the metadata to the user's prompt as `key: value` lines wrapped in
markers, before the LLM runs:

```
How do I set up SSO?

<CONTEXT METADATA> Here is additional context metadata for this conversation:
product: Analytics
planTier: Enterprise
region: EU
</END CONTEXT METADATA>
```

The agent only *acts* on this if its **system prompt** tells it to. To make behavior track
the context, reference the keys explicitly, e.g. "When `planTier` is given, only suggest
features on that plan; when `region` is EU, note data-residency." Unlike `page_content`,
these markers are **not** stripped — they remain in saved chat history.

## Per transport

Same `metadata` field, same pipeline, every transport:

- **SSE / WebSocket** — a field in the `BaseConsumerPayload` body (`SKILL.md ## Writes`);
  identical shape at `POST …/api/agent/chat/` and `wss://asgi.data.iblai.app/ws/chat/`.
- **Embedded iframe** — the host page has no direct socket, so it hands context to the
  chat widget over `postMessage`; the widget forwards it into its WS payload:
  ```javascript
  iframe.contentWindow.postMessage({
    type: 'MENTOR:CONTEXT_UPDATE',         // verbatim protocol constant
    hostInfo: { title: document.title, href: location.href },
    pageContent: bodyText,                  // → page_content
    metadata: { productGroup: 'LICENSING', stateCode: 'CA' }
  }, '*');
  ```
  Widget contract: listen for `type: 'MENTOR:CONTEXT_UPDATE'`, take `metadata`, include it
  on every `/ws/chat/` send.

## Session semantics

- **Send once** — cached on the session, reused for later turns that omit it.
- **Replace, not merge** — a new `metadata` object overwrites the cached one whole.
- **Null / omitted = no change** — the last value keeps applying.
- Cache is short-lived (~2h TTL) but also written to the session record, so it survives
  cache expiry; reads always reflect the persisted `client_context`.

## Storage & pipeline

The validated payload (`BaseConsumerPayload`) reaches the consumer
(`BaseLLMRunnerConsumer.process_text_data`), which caches *and* persists the metadata; the
runner (`LLMRunner.asetup_user_prompt`) then composes the prompt in order — greeting
instructions, then any `page_content`, then the metadata block — and the message is saved
with the markers intact. It lands in three places:

| Layer | Location | Role |
|-------|----------|------|
| Cache | Redis `session_{session_id}_metadata` | fast access during the live conversation (~2h TTL) |
| Session | `Session.metadata["client_context"]` | persistent store the reads and exports return |
| Per message | `ChatMessageHistoryExtra.metadata` | snapshot of the metadata as it stood at each message |

The session layer is what `client_context` reads back; the per-message layer additionally
records the metadata *as it was at that message*, so exported history reflects mid-session
changes. Beyond the reads in `SKILL.md`, the same value also surfaces as a `client_context`
column in the analytics reporting export (`get_chat_message_history`).

## One agent, many contexts

The point of session-level metadata: a single deployed agent serves every surface and the
caller sets the frame per session. The Analytics page on Enterprise sends
`{product:"Analytics", planTier:"Enterprise", region:"EU"}`, and "How do I set up SSO?"
gets Enterprise SSO steps with EU data-residency notes; the Payments page on Starter sends
`{product:"Payments", planTier:"Starter"}`, and the *same* agent explains Starter
integration and that SSO needs an upgrade. No per-surface agents, no prompt edits — just
the metadata plus a system prompt that reads it.
