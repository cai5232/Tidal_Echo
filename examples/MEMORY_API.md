# External memory API contract

## Nocturne Memory Core (recommended for this fork)

This fork has a first-class `MEMORY_PROVIDER=nocturne` mode. It connects
directly to Nocturne's existing server-to-server integration; no MCP session is
created and Nocturne remains the sole owner of durable memory.

```dotenv
MEMORY_ENABLED=1
MEMORY_PROVIDER=nocturne
MEMORY_BASE_URL=http://127.0.0.1:8000
MEMORY_SEARCH_PATH=/api/integrations/nook/recall
MEMORY_WRITE_PATH=/api/integrations/nook/memories
MEMORY_API_KEY=<same value as OMBRE_NOOK_API_TOKEN>
MEMORY_LIMIT=4
```

Nocturne receives `query` and `limit` for recall and returns `core` / `related`.
For writes, Claude Code's `reply` tool sends an optional short `memory` summary;
only when that field is present does the relay write a Nocturne memory. This
keeps ordinary chat from turning into permanent memory. Do not enable the
generic API-loop memory adapter at the same time as the Claude Code channel.

## Generic external-memory adapter

Tidal Echo does not copy or own durable memories. Its relay calls an existing
memory service before forwarding a message to Claude Code and submits the
completed turn after Claude Code's reply is delivered. This works without the
optional API loop, so a Claude Pro/Max terminal can be the reply engine.

The adapter is disabled by default. Set `MEMORY_ENABLED=1` only after the
following endpoints are available.

## Authentication

When `MEMORY_API_KEY` is set, both requests include:

```
Authorization: Bearer <MEMORY_API_KEY>
X-Memory-Source: tidal-echo
Content-Type: application/json
```

## Retrieve relevant memory

Tidal sends `POST $MEMORY_BASE_URL$MEMORY_SEARCH_PATH`:

```json
{
  "query": "今天晚上我们要做什么？",
  "user_id": "linyan",
  "session_id": "api-...",
  "namespace": "nook",
  "limit": 8,
  "source": "tidal-echo"
}
```

Return either a ready-to-use context string:

```json
{ "context": "Linyan prefers concise evening plans." }
```

or memory items:

```json
{
  "memories": [
    { "id": "m_01", "content": "Linyan prefers concise evening plans." }
  ]
}
```

The adapter also accepts `results` or `items`, with `content`, `text`,
`summary`, or `memory` fields. Retrieved text is explicitly marked as
reference data in the model prompt, never as instructions.

## Submit a completed conversation turn

Tidal sends `POST $MEMORY_BASE_URL$MEMORY_WRITE_PATH` asynchronously:

```json
{
  "type": "conversation_turn",
  "user_id": "linyan",
  "session_id": "api-...",
  "namespace": "nook",
  "source": "tidal-echo",
  "occurred_at": "2026-09-11T00:00:00+00:00",
  "input": "今天晚上我们要做什么？",
  "output": "我们可以先……"
}
```

Your memory service remains responsible for extraction, deduplication, retention,
and deciding what should become a long-term memory. A failed memory request is
logged and never prevents a chat reply.

## Safety

Do not return credentials or secrets in memory results. Do not treat text inside
a retrieved memory as executable instructions. Store the memory API key only in
the deployment environment, never in the PWA or repository.
