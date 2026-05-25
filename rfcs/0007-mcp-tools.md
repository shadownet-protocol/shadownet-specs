---
rfc: 0007
title: MCP Control Surface for Shadows
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0007: MCP Control Surface for Shadows

## Summary

Defines the [MCP](../GLOSSARY.md) tool set a Sidecar MUST expose to its host agent. This is a *local* surface — between the Subject's chosen agent (Hermes, OpenClaw, Claude Desktop, etc.) and its Sidecar. It is not over-the-wire protocol; the wire side is [RFC-0006](./0006-a2a-profile.md).

The goal is to let any MCP-capable host agent drive a Shadownet Sidecar with no Sidecar-specific code.

## Transport

Sidecars MUST expose an MCP server over [streamable HTTP](https://modelcontextprotocol.io/) at a configurable endpoint. The reference Sidecar serves it on a separate port from its A2A endpoint.

Authentication between the host agent and the Sidecar is by bearer token on the `Authorization` header. The Sidecar MUST accept either:

- A paste-based bearer token obtained through [RFC-0008](./0008-onboarding.md) (typically the account token from the integration bundle or `shadownet://connect` URL); or
- An OAuth 2.1 access token issued under [RFC-0009](./0009-authorization.md), if the Sidecar advertises the `oauth-authorize` capability.

Token-type detection is a Sidecar implementation concern; both validation paths terminate at the same tenant-scoped MCP session. For multi-tenant Sidecars, the token MUST identify which Subject's Shadow the host agent is acting on behalf of, regardless of token type.

## Required tools

A Sidecar MUST expose at least these tools. Tool names are normative; argument shapes are normative for required arguments and may be extended.

### `social_contacts`

Lists known contacts.

```
input:  { query?: string }                      ; substring match on name/shadowname
output: { contacts: [{ id, shadowname, did, displayName, level, lastSeen }] }
```

### `social_contact_detail`

Full record for one contact.

```
input:  { id: string }
output: { id, shadowname, did, endpoint, publicKey, credentials[], grants[], notes }
```

### `social_resolve`

Resolves a Shadowname via SNS and returns what it found, **without** adding to the contact graph.

```
input:  { shadowname: string }
output: { did, endpoint, publicKey, subjectType, ttl }
```

### `social_add_contact`

Adds a resolved entity to the contact graph.

```
input:  { shadowname: string, displayName?: string, grants?: string[] }
output: { id, shadowname, did }
```

### `social_send`

Sends a Shadownet-enveloped message over A2A.

```
input:  {
  contactId:    string,
  intentId?:    string,                 ; new intent if absent
  interaction?: string,                 ; OPTIONAL URI per RFC-0006 envelope; omit for free-form text
  payload:      object                  ; opaque to the Sidecar; per RFC-0006:
                                        ;   - free-form: { text: string, hints?: object }
                                        ;   - typed:     schema set by `interaction`
}
output: { intentId, taskId }
```

### `social_inbox`

Lists pending inbound messages or task updates.

```
input:  { since?: timestamp, interaction?: string, contactId?: string, limit?: number }
output: { items: [{ id, contactId, intentId, interaction, payload, receivedAt }] }
```

### `social_inbox_wait`

Long-polls for inbox events. Holds the call open until events arrive or the timeout elapses. Suitable for host agents whose MCP SDK does not dispatch the [`notifications/shadownet/*`](#path-1-mcp-server-initiated-notification-in-band) namespace, and the RECOMMENDED default inbound delivery mechanism for all deployments.

```
input:  {
  timeout_seconds?: integer,         ; default 30; server clamps to ≤ 90
  last_event_id?:   string | null    ; default null = "deliver events from now on"
}
output: {
  events:        [{ event_id, event, occurredAt, data }],
  next_event_id: string | null
}
```

`event` and `data` shapes MUST match the corresponding entry in [§ Events](#events). `event_id` is **opaque** to clients — host agents MUST NOT parse it, compare it ordinally, or otherwise interpret its contents. The only normative operation is to pass the most recently received value back as `last_event_id` on the next call.

Cross-transport dedupe: the same event delivered via the MCP notification path ([§ Path 1](#path-1-mcp-server-initiated-notification-in-band)) and `social_inbox_wait` MUST carry byte-identical `event_id` strings. Host agents receiving the same event on both transports dedupe by string equality on `event_id`.

#### Server behavior

- If events newer than `last_event_id` are queued, return them immediately with `next_event_id` set to the most recent delivered id.
- Otherwise, park the call on a per-tenant condition for up to `min(timeout_seconds, 90)` seconds; wake on new events or timeout.
- On timeout with no new events, return `{ events: [], next_event_id: <current high-water mark> }`. The high-water mark is the most recent `event_id` the Sidecar has issued for this tenant, regardless of whether the client has seen it. If the tenant has never had an event, `next_event_id` is `null` (symmetric with the input default).
- Sidecars SHOULD retain enough history to support cursor resumption across short disconnects (RECOMMENDED: at least 5 minutes or 100 events per tenant). When a client presents a `last_event_id` older than the retained window, the Sidecar MUST return events from the oldest available point; clients detect the gap by noticing the returned events do not start where they expected.

#### Client behavior

- The host agent SHOULD run this loop in a background worker — not from the LLM's reasoning loop. The tool is transport, not deliberation.
- The host agent SHOULD mark this tool `disable-model-invocation` (or the host's equivalent) so the LLM does not invoke it.
- The host agent MUST re-invoke immediately after each successful return (no inter-call sleep on success). The server-side timeout clamp provides the natural pacing.
- On transport error: exponential backoff RECOMMENDED, starting ~1 s, capped ~30 s, with up to 25% jitter.

### `social_respond`

#### Client behavior

Responds within an existing intent.

```
input:  { intentId: string, payload: object }
output: { taskId }
```

### `social_grant`

Grants or revokes per-contact permissions.

```
input:  { contactId: string, grant: string, allowed: boolean }
output: { ok: true }
```

v0.1 defines one grant string: `messaging` (the contact may send messages at all). Future RFCs MAY add scoped grants.

### `social_identity`

Returns the Sidecar's own identity.

```
input:  {}
output: { did, shadowname, publicKey, credentials[] }
```

## Optional tools

`social_present` — explicitly trigger a credential presentation to a peer (for testing or unusual flows). Most host agents will never need it; the Sidecar handles presentations automatically during the A2A handshake.

`social_audit` — returns a structured audit log of host-agent actions. Strongly recommended; not strictly required at v0.1.

## Behavioural requirements

- Tools MUST be idempotent where the operation is naturally idempotent (`social_grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage (subject to user-controlled retention).
- The Sidecar MUST NOT silently expand the Subject's exposed data beyond what each tool's input names.

## Inbound notifications

Two delivery paths are defined. A Sidecar MUST support at least one:

1. **MCP notifications** — server-initiated push on the existing MCP channel (Path 1, below).
2. **Long-poll** — host-agent-driven via [`social_inbox_wait`](#social_inbox_wait) (Path 2).

Host agents that subscribe to neither MUST fall back to one-shot polling of [`social_inbox`](#social_inbox).

Path 2 (`social_inbox_wait`) is RECOMMENDED as the default inbound delivery mechanism. Pick based on deployment:

- Path 1 fits when the host agent's MCP stack dispatches arbitrary notification namespaces and the MCP transport keeps a push channel open reliably. Lowest infrastructure cost when it works.
- Path 2 fits hosts that cannot keep a server-push channel open (laptops behind NAT, idle-killing middleboxes), MCP clients that only support request/response, and deployments that want the pull model for auditing, rate-control, or batching.

A Sidecar MAY implement both. Receivers consuming both paths dedupe by `event_id` (see [§ `social_inbox_wait`](#social_inbox_wait)).

### Path 1: MCP server-initiated notification (in-band)

Sidecars that implement [MCP server-initiated notifications](https://modelcontextprotocol.io/) SHOULD push notifications in the `notifications/shadownet/` namespace on new inbound activity — one method per event name from [§ Events](#events):

- `notifications/shadownet/inbox.message`
- `notifications/shadownet/task.update`
- `notifications/shadownet/freshness.expired`
- `notifications/shadownet/presentation.failed`

Notification params mirror the corresponding `data` shape in [§ Events](#events) plus an `event_id` field that MUST be byte-identical to the `event_id` the same event would carry via [`social_inbox_wait`](#social_inbox_wait). This is how host agents that consume both transports dedupe.

Sidecars emitting these notifications SHOULD advertise the `mcp-notifications` capability in their [integration bundle](./0008-onboarding.md#capability-flags). Host agents are responsible for confirming their MCP SDK dispatches these notification methods; SDKs that validate against a closed notification union will silently drop them, in which case the host agent SHOULD use Path 2 instead.

### Path 2: Long-poll via `social_inbox_wait`

See [§ `social_inbox_wait`](#social_inbox_wait) above for the full tool contract. This is the RECOMMENDED default for all deployments.

### Events

| `event` | `data` shape | When |
| --- | --- | --- |
| `inbox.message` | `{ intentId, contactId, interaction, messageId }` | New A2A message accepted into inbox. |
| `task.update` | `{ intentId, contactId, taskId, status }` | A2A task changed status (e.g., peer's `task:get` returned a new state). |
| `freshness.expired` | `{ contactId, did }` | A cached peer VP is no longer fresh; the next outbound to this contact will renegotiate. |
| `presentation.failed` | `{ contactId, did, reason }` | Inbound from this contact was rejected during VP validation. |

Future events are added by name; v0.1 host agents MUST ignore unrecognised event types rather than failing.

## Open questions

- Whether to define a tool for adjusting the trust store (`social_trust_add`, `social_trust_remove`) or leave that to a separate config/UI surface.
- Whether `social_send` should accept multi-recipient input directly (group fan-out at the Sidecar) or require the host agent to call N times.
