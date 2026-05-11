# Tool schemas

JSON Schemas for MCP-tool inputs and outputs. Each schema is referenced from the RFC that defines the tool (today: [RFC-0007 § Required tools](../../rfcs/0007-mcp-tools.md#required-tools)).

## Naming

- `<tool-name>-args.schema.json` — input shape (the `input:` block in the RFC).
- `<tool-name>-result.schema.json` — output shape (the `output:` block in the RFC).

`<tool-name>` is the MCP tool name with `_` replaced by `-` (kebab-case, matching the rest of `schemas/`). The tool `social_inbox_wait` maps to `social-inbox-wait-result.schema.json`.

A tool needs an entry here only when its shape is non-trivial — purely scalar or empty shapes (`{ ok: true }`, `{}`) do not. Add an args schema only when the input has structure worth validating; many tools will ship with only a result schema, or only an args schema, and that is fine.

## `$id`

`https://sh4dow.org/schemas/v<MAJOR>/tools/<filename>` — matches the top-level convention in [`../README.md`](../README.md).
