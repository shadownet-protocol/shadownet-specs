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

Authentication between the host agent and the Sidecar is local; v0.1 uses a bearer token issued at Sidecar startup or signup. (For multi-tenant cloud Sidecars, the token authenticates *which* Subject's Shadow the host agent is acting on behalf of.)

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
  contactId:   string,
  interaction: string,                 ; URI per RFC-0006 envelope
  intentId?:   string,                 ; new intent if absent
  payload:     object                  ; opaque to the Sidecar
}
output: { intentId, taskId }
```

### `social_inbox`

Lists pending inbound messages or task updates.

```
input:  { since?: timestamp, interaction?: string, contactId?: string, limit?: number }
output: { items: [{ id, contactId, intentId, interaction, payload, receivedAt }] }
```

### `social_respond`

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

### `social_set_webhook`

Registers (or updates) the host-agent webhook to which the Sidecar pushes inbound activity. See [Inbound notifications](#inbound-notifications) for the wire contract.

```
input:  {
  url:     string,                 ; HTTPS or http://localhost only
  secret:  string,                 ; ≥32 bytes, host-agent-chosen, used for HMAC
  events?: string[]                ; default: all events
}
output: { ok: true }
```

To unregister, call with `url: ""`. The Sidecar persists the registration and replays it after restart.

## Optional tools

`social_present` — explicitly trigger a credential presentation to a peer (for testing or unusual flows). Most host agents will never need it; the Sidecar handles presentations automatically during the A2A handshake.

`social_audit` — returns a structured audit log of host-agent actions. Strongly recommended; not strictly required at v0.1.

## Behavioural requirements

- Tools MUST be idempotent where the operation is naturally idempotent (`social_grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage (subject to user-controlled retention).
- The Sidecar MUST NOT silently expand the Subject's exposed data beyond what each tool's input names.

## Inbound notifications

Two push paths are defined; a Sidecar MUST support at least one. Hosts that subscribe to neither MUST poll `social_inbox`.

### Path 1: MCP server-initiated notification (in-band)

Sidecars that implement [MCP server-initiated notifications](https://modelcontextprotocol.io/) SHOULD send a `shadownet/inbox` notification on new inbound activity. Payload mirrors the webhook event payload (below) minus the HMAC headers.

### Path 2: Webhook (out-of-band)

For host agents that prefer (or only support) HTTP webhooks. Registered via [`social_set_webhook`](#social_set_webhook).

#### Wire shape

```http
POST <registered url> HTTP/1.1
Content-Type: application/json
X-Shadownet-Sidecar-Sig: sha256=<hex HMAC-SHA256 of body, key=secret>
X-Shadownet-Sidecar-Ts:  1759200200
X-Shadownet-Sidecar-Id:  <opaque Sidecar instance id>

{
  "shadownet:v": "0.1",
  "event":       "inbox.message",
  "occurredAt":  1759200200,
  "data": {
    "intentId":    "urn:uuid:int-001",
    "contactId":   "ctc_sarah01",
    "interaction": "urn:shadownet:int:scheduling.v0-draft",
    "messageId":   "msg-..."
  }
}
```

The webhook payload deliberately does NOT carry the message *content* — the host agent is expected to call `social_inbox` for the actual payload. This keeps the webhook small, replay-safe, and avoids leaking content into webhook logs.

#### Receiver requirements

The host agent MUST:

1. Verify the HMAC. Compare `X-Shadownet-Sidecar-Sig` (after stripping the `sha256=` prefix) against `HMAC-SHA256(secret, body)`.
2. Reject deliveries whose `X-Shadownet-Sidecar-Ts` differs from local time by more than 5 minutes (replay defense).
3. Respond `2xx` on accepted delivery; any other status (including `5xx`) triggers retry.
4. Be idempotent on `messageId` (a delivery may arrive more than once due to retries).

#### Retries

If delivery fails (timeout > 10 s, non-2xx, connection error), the Sidecar SHOULD retry with exponential backoff:

| Attempt | Delay from previous |
| --- | --- |
| 1 | immediate |
| 2 | +5 s |
| 3 | +30 s |
| 4 | +5 min |
| 5 | +30 min |

After attempt 5, the Sidecar SHOULD mark the webhook as **degraded** and continue serving the host agent via polling. Delivery resumes when `social_set_webhook` is called again or the host agent successfully calls any tool (treated as a liveness signal).

#### URL constraints

`url` MUST be `https://…` OR `http://localhost…` / `http://127.0.0.1…` / `http://[::1]…`. All other `http://` schemes MUST be rejected (`invalid_webhook_url`). This prevents accidental plaintext deliveries off-host.

#### Events

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
- Whether webhook secrets should be rotatable without re-registration.
