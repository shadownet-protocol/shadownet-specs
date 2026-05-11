# Well-known host slugs

The set of `<host>` values a Sidecar may serve from [`<base>/connect/<host>`](../rfcs/0008-onboarding.md#per-host-install-pages).

**Non-normative.** This file is a living registry. Adding, removing, or updating a host is a PR to *this file only* — no RFC amendment, no version bump. The RFC defines the route shape and the content-negotiation rules; everything below is a coordination point so Sidecar implementations and host-agent docs agree on the same slug spellings.

## Registry

| Slug | Host | Snippet MUST contain |
| --- | --- | --- |
| `hermes-agent` | [Hermes Agent](https://hermes-agent.nousresearch.com/) (Nous Research) | The plugin-install incantation and the env-var exports the Hermes plugin expects (e.g., `SHADOWNET_TOKEN`, `SHADOWNET_SIDECAR_BASE_URL`). |
| `claude-code` | [Claude Code](https://claude.com/claude-code) (Anthropic) | A `claude plugins install` command and an `.mcp.json` block referencing the Sidecar's MCP endpoint and bearer token. |
| `openclaw` | OpenClaw | An `openclaw plugins install` command and the `configSchema` answers the plugin's prompts expect. |
| `cursor` | [Cursor](https://cursor.com) | The MCP server JSON block to paste into Cursor's settings, with URL + auth header pre-filled. |
| `continue` | [Continue](https://continue.dev) | The Continue YAML config block for the Sidecar's MCP endpoint. |
| `raw` | Universal escape hatch | The canonical integration bundle JSON, unchanged from `/v1/account/me/integration-bundle`. Reserved slug — implementations MUST NOT register a host under this name. |

## Content shapes

For each slug, the Sidecar's response varies by `Accept` header per [RFC-0008 § Content negotiation](../rfcs/0008-onboarding.md#content-negotiation). Notes per host:

- `hermes-agent`, `openclaw`: `application/json` returns `{ "configSchema": { ... } }` — the same shape the plugin's CLI prompts answer.
- `claude-code`, `cursor`: `application/json` returns `{ "mcpServerConfig": { ... } }` — a single MCP server entry the host can drop into its settings file.
- `continue`: `application/json` returns `{ "yaml": "<rendered yaml>" }`. (YAML in a JSON string is awkward; Continue's config is YAML-only today.)
- `raw`: `application/json` is the canonical bundle. `text/plain` and `text/html` are RECOMMENDED but not required.

## Adding a host

1. Open a PR adding a row to the registry table and an entry to § Content shapes if its JSON form differs from the others.
2. Confirm at least one Sidecar implementation can serve the slug end-to-end before merge.
3. No spec change, no version bump, no GLOSSARY update.

If a host requires a normative *protocol* change to be installable (a new MCP tool, a new event, an envelope extension), that change belongs in the relevant RFC — not here.
