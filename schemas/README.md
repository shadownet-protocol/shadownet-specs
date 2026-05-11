# Schemas

JSON Schema draft 2020-12. Each schema is referenced from at least one RFC.

## Layout

- `messages/` — A2A envelope and message-shape schemas.
- `credentials/` — VC payload and freshness-proof schemas.
- `events/` — (placeholder) audit / log events.
- `onboarding/` — integration bundle and connect-URL shapes (RFC-0008).
- `tools/` — MCP tool I/O schemas (e.g. `social_inbox_wait` result).

## Naming

`<concept>.schema.json` (e.g. `subject-credential.schema.json`).

## `$id`

`https://sh4dow.org/schemas/v<MAJOR>/<path>.schema.json`.
