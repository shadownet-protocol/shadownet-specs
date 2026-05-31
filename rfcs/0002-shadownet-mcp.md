---
rfc: 0002
title: Shadownet MCP Control Surface
version: v0.2
shadownet: urn:shadownet:0.2
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet MCP Control Surface

## 1. Summary

Defines the MCP tool set a Sidecar MUST expose to its host LLM. This is a local surface between the Subject's host LLM and its Sidecar; the wire side is [RFC 0001 Shadownet](./0001-shadownet.md).

This document is **content-agnostic.** It defines the control plane — contact graph operations, envelope dispatch, inbox access — without prescribing what conversations look like. Application-level interaction schemas (scheduling, intro, payment, structured negotiation, etc.) are defined in separate intent profile companion specs and reach the host LLM through the opaque `body.intent` / `body.data` slots on the generic `send` and `inbox` tools.

## 2. Transport

Sidecars MUST expose an MCP server over [streamable HTTP](https://modelcontextprotocol.io/) at a configurable endpoint.

Authentication is by bearer token on the `Authorization` header. Token issuance is the [onboarding companion](./0003-shadownet-onboarding.md)'s concern; the token is opaque to the host LLM and identifies which Subject's Shadow the host is acting for. A Sidecar MUST reject requests with missing or invalid tokens using MCP's standard error mechanism.

## 3. Conventions

Tool names use **snake_case**. JSON argument and result field names use **camelCase** (Shadownet [RFC 0001 §2 naming table](./0001-shadownet.md#2-conventions)). Value strings use **snake_case**. Event names use dotted lowercase.

Identifiers in tool inputs and outputs accept either a Shadowname (`alice@sh4dow.org`), a direct connection URI (`shadow://key:z6Mk...@host:port`), or a bare key (multibase Ed25519) — whichever form the contact uses. Sidecars resolve the form internally and store contacts under their wire-internal identifier (Shadowname or bare key) per RFC 0001 §3.3. The token authenticates the Subject; no `subject` parameter is carried on individual calls.

Body content (`body.intent`, `body.data`) is **opaque to this surface**. Sidecars MUST pass body content unchanged between the host LLM and the envelope. Schema validation against known intent profiles is OPTIONAL and is performed at the application layer, not at the MCP control surface.

## 4. Required tools

A Sidecar MUST expose at least these tools. Tool names are normative; argument shapes are normative for required arguments and may be extended.

### `identity`

Returns the Sidecar's own identity, including both addressing forms if available.

```
input:  {}
output: {
  shadowname?: string,                    // present iff the Subject has Shadowname-mode addressing
  directUri?:  string,                    // present iff the Subject has direct-mode addressing
  pk:          string,                    // multibase Ed25519
  credentials: [
    { kind:      "org_affiliation",
      issuer:    string,
      org:       string,
      expiresAt: string                   // ISO 8601
    }
  ]
}
```

A Subject MAY have one or both addressing forms (e.g., a keyed Shadow that later registered a Shadowname has both). At least one of `shadowname` or `directUri` MUST be present.

### `resolve`

Resolves an identifier via Shadownet RFC 0001 §5 without adding to the contact graph. Accepts a Shadowname or a `shadow://` URI.

```
input:  { name: string }
output: { identifier: string, pk: string, endpoint: string }
error:  "resolve_failed" | "unreachable"
```

`identifier` is the canonical wire-internal form (Shadowname or bare key).

### `contacts`

Lists known contacts.

```
input:  { query?: string }                 // substring match on identifier or displayName
output: { contacts: [
  { identifier: string,                    // Shadowname or bare key
    displayName?: string,
    grants:      string[],
    lastSeen?:   string                    // ISO 8601
  }
]}
```

### `contact_detail`

Full record for one contact.

```
input:  { name: string }
output: {
  identifier:  string,
  displayName?: string,
  pk:          string,
  endpoint:    string,
  grants:      string[],
  credentials: [
    { kind: string, issuer: string, expiresAt: string }
  ],
  profile?:    ContactProfile,             // §5
  addedAt:     string,
  lastSeen?:   string,
  tlsPin?:     string                       // sha256 fingerprint; present for direct-mode contacts
}
error:  "not_contact"
```

### `add_contact`

Adds an identifier to the contact graph. Held stranger-review items from this identifier MUST be graduated to inbox after the contact is added.

```
input:  {
  name:         string,                    // Shadowname or shadow:// URI
  displayName?: string,
  grants?:      string[],                  // defaults to ["messaging"]
  profile?:     ContactProfile             // §5
}
output: {
  identifier:    string,
  trustWarning?: { untrustedIssuers: string[] }   // present iff cached credentials reference issuers not in the trust store
}
error:  "resolve_failed" | "already_contact"
```

Per Shadownet RFC 0001 §7.1, Sidecars ship with an empty trust store. Adding a contact whose credentials reference an untrusted issuer succeeds; the `trustWarning` field surfaces the issuer for the host LLM to relay to the user. Trust-store edits themselves are not exposed through this surface (§8).

### `grant`

Sets or clears a per-contact permission.

```
input:  { name: string, grant: string, allowed: boolean }
output: { ok: true }
error:  "not_contact" | "unknown_grant"
```

Defined grant: `messaging` (the contact's envelopes reach `inbox`; clearing it routes future inbound to `stranger_review` per Shadownet RFC 0001 §9). Future revisions MAY add grants by string.

### `set_contact_profile`

Updates the local-only profile on a contact. Passing `{}` clears all fields. Partial updates are not defined; clients SHOULD read the current profile via `contact_detail` and submit the full desired state.

```
input:  { name: string, profile: ContactProfile }
output: { ok: true }
error:  "not_contact"
```

### `send`

Sends a Shadownet envelope. The `body` is passed opaquely to the envelope per RFC 0001 §8.5; this surface does not interpret `body.intent` or `body.data`.

```
input:  {
  to:         string,                      // recipient identifier (Shadowname or shadow:// URI)
  body: {                                  // per RFC 0001 §8.5
    text?:    string,
    intent?:  string,
    data?:    object
  },
  contextId?: string                       // new context if absent
}
output: {
  messageId: string,
  contextId: string,
  status:    "accepted" | "rejected",
  error?:    string                         // Shadownet error code (URN suffix); present iff status=="rejected"
}
```

Per Shadownet RFC 0001 §9, replies arriving on the same `contextId` are auto-added to the contact graph on both sides; host LLMs do not need to call `add_contact` for replies to outbound messages.

### `respond`

Convenience wrapper around `send` for continuing an existing thread. Equivalent to `send` with the named `contextId`.

```
input:  { contextId: string, body: { text?, intent?, data? } }
output: { messageId, status, error? }
```

### `inbox`

Lists pending inbound messages with full body content.

```
input:  {
  since?:         string,                  // opaque cursor
  contact?:       string,                  // filter to envelopes from this identifier
  intent?:        string,                  // filter to envelopes matching this intent URI
  includeReview?: boolean,                 // default false
  limit?:         integer                  // default 50
}
output: {
  items: [
    { messageId:  string,
      contextId:  string,
      from:       string,
      receivedAt: string,
      status:     "inbox" | "stranger_review",
      body: { text?: string, intent?: string, data?: object }
    }
  ],
  nextSince: string | null
}
```

The cursor `since` is opaque — host LLMs MUST NOT parse or compare it ordinally. Pass `nextSince` from one call as `since` in the next.

The `intent` filter is provided so host LLMs that have opted into a specific intent profile can subscribe to envelopes of that profile without scanning all inbox content. The Sidecar performs string-equality match on the envelope's `body.intent`; no schema interpretation.

### `inbox_wait`

Long-polls for inbox events. RECOMMENDED default inbound delivery mechanism.

```
input:  {
  timeout_seconds?: integer,               // default 30; server clamps to ≤ 90
  last_event_id?:   string | null          // default null = "deliver events from now on"
}
output: {
  events:        [ { eventId, event, occurredAt, data } ],
  next_event_id: string | null
}
```

`event` and `data` shapes MUST match the corresponding entry in §6. `eventId` is opaque — host LLMs MUST NOT parse it; pass the most recent back as `last_event_id` on the next call.

Cross-transport dedupe: the same event delivered via Path 1 (§6) and `inbox_wait` MUST carry byte-identical `eventId` strings.

#### Server behavior

- If events newer than `last_event_id` are queued, return them immediately with `next_event_id` set to the most recent delivered id.
- Otherwise park up to `min(timeout_seconds, 90)` seconds; wake on new events or timeout.
- On timeout with no new events, return `{ events: [], next_event_id: <current high-water mark> }`. If the tenant has never had an event, `next_event_id` is `null`.
- Sidecars SHOULD retain enough history to support cursor resumption across short disconnects (RECOMMENDED: at least 5 minutes or 100 events per tenant).

#### Client behavior

- The host LLM SHOULD run this loop in a background worker, not from the LLM reasoning loop.
- The host LLM SHOULD mark this tool `disable-model-invocation` (or the host's equivalent).
- The host LLM MUST re-invoke immediately after each successful return; the server-side timeout clamp provides the pacing.
- On transport error: exponential backoff RECOMMENDED, starting ~1 s, capped ~30 s, with ±25% jitter.

## 5. ContactProfile

Local-only metadata the Subject attaches to a contact, persisted by the Sidecar and surfaced via `contact_detail`.

A ContactProfile MUST NOT be transmitted to the contact or any other peer. It MUST NOT appear in any over-the-wire artifact (envelope, AgentCard, credential, status list).

### Shape

```json
{
  "notes":     "Contractor on Project Foo. Prioritize his messages.",
  "priority":  "high",
  "tags":      ["project_foo", "external"],
  "expiresAt": "2026-08-01T00:00:00Z"
}
```

All fields OPTIONAL:

| Field | Type | Purpose |
| --- | --- | --- |
| `notes` | string | Free-form text. Maximum 4 KiB. |
| `priority` | enum: `low` \| `normal` \| `high` | Routing hint. Default `normal`. |
| `tags` | string array | Categories (project codenames, relationship type, affiliation, etc.). |
| `expiresAt` | RFC 3339 timestamp | Auto-archive date for time-bounded relationships. |

## 6. Inbound notifications

Two delivery paths are defined. A Sidecar MUST support at least one:

1. **MCP notifications** (Path 1) — server-initiated push on the existing MCP channel.
2. **Long-poll** (Path 2) — host-LLM-driven via `inbox_wait`.

Path 2 is the RECOMMENDED default. Host LLMs that subscribe to neither MUST fall back to one-shot polling of `inbox`. A Sidecar MAY implement both; receivers consuming both paths dedupe by `eventId`.

### Path 1: MCP server-initiated notification

Sidecars SHOULD push in the `notifications/shadownet/` namespace on new inbound activity:

- `notifications/shadownet/inbox.message`
- `notifications/shadownet/task.update`

Notification params mirror the corresponding `data` shape in the events table below plus an `eventId` field byte-identical to the `eventId` the same event would carry via `inbox_wait`.

### Path 2: Long-poll via `inbox_wait`

See `inbox_wait` (§4).

### Events

| `event` | `data` shape | When |
| --- | --- | --- |
| `inbox.message` | `{ messageId, contextId, from, intent?, status }` | New envelope accepted into `inbox` or `stranger_review`. Body is fetched via `inbox`. |
| `task.update` | `{ contextId, taskId, status }` | An A2A task changed status. Only emitted for application-opened A2A `Task` workflows; Shadownet's standard envelope responses use A2A `Message` and do not generate this event. |

Host LLMs MUST ignore unrecognised event types rather than failing.

## 7. Behavioural requirements

- Tools MUST be idempotent where the operation is naturally so (`grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage, subject to user-controlled retention.
- The Sidecar MUST NOT return data beyond what each tool's contract names.

## 8. Out of scope

- **Application intent profiles** (scheduling, intro, payment, structured negotiation, etc.). Each intent profile is defined in its own companion spec (RFC 0004+). The MCP surface here is intentionally content-agnostic; intent profiles reach the host LLM through the opaque `body.intent` / `body.data` slots on `send` / `inbox` and are interpreted at the application layer.
- **Trust-store editing.** Not reachable from the host LLM's tool surface; belongs in the Sidecar's account portal.
- **Audit endpoints.** Sidecars MAY expose audit logs over a separate surface; format is not standardized here.
- **OAuth-style scoped tokens.** Bearer tokens grant full access; deployments needing scopes issue separate tokens per scope.