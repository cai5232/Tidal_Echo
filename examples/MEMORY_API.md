# External memory API contract

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
