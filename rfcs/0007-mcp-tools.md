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

## Optional tools

`social_present` — explicitly trigger a credential presentation to a peer (for testing or unusual flows). Most host agents will never need it; the Sidecar handles presentations automatically during the A2A handshake.

`social_audit` — returns a structured audit log of host-agent actions. Strongly recommended; not strictly required at v0.1.

## Behavioural requirements

- Tools MUST be idempotent where the operation is naturally idempotent (`social_grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage (subject to user-controlled retention).
- The Sidecar MUST NOT silently expand the Subject's exposed data beyond what each tool's input names.

## Notification surface

A Sidecar SHOULD expose an MCP **server-initiated notification** when a new inbox item arrives, so a host agent can react without polling. v0.1 hosts that don't subscribe MUST poll `social_inbox`.

## Open questions

- Whether to define a tool for adjusting the trust store (`social_trust_add`, `social_trust_remove`) or leave that to a separate config/UI surface.
- Whether `social_send` should accept multi-recipient input directly (group fan-out at the Sidecar) or require the host agent to call N times.
- Standard tool for "summarize what my Shadow has been doing today" — useful for auditability but maybe orthogonal to MCP.
