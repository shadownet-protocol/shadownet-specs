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

Defines the MCP tool set a Sidecar MUST expose to its host LLM. This is a *local* surface — between the Subject's chosen host LLM (Hermes, OpenClaw, Claude Desktop, etc.) and its Sidecar. It is not over-the-wire protocol; the wire side is [RFC 0001 Shadownet](./0001-shadownet.md).

The goal is to let any MCP-capable host LLM drive a Shadownet Sidecar with no Sidecar-specific code.

## 2. Transport

Sidecars MUST expose an MCP server over [streamable HTTP](https://modelcontextprotocol.io/) at a configurable endpoint.

Authentication between the host LLM and the Sidecar is by bearer token on the `Authorization` header. Token issuance is the [onboarding companion](./0003-shadownet-onboarding.md)'s concern; the token is opaque to the host LLM and identifies which Subject's Shadow the host is acting for.

A Sidecar MUST reject requests with missing or invalid tokens using MCP's standard error mechanism.

## 3. Conventions

Tool names use **snake_case** (MCP idiom). JSON argument and result field names use **camelCase** (Shadownet [RFC 0001 §2 naming table](./0001-shadownet.md#2-conventions)). Value strings (statuses, kinds, grant names, intent URI suffixes) use **snake_case**. Event names use dotted lowercase.

Shadowname strings appearing in arguments and results are canonical lowercase per Shadownet RFC 0001 §5.1. The token authenticates the Subject; no `subject` parameter is carried on individual calls.

## 4. Required tools

A Sidecar MUST expose at least these tools. Tool names are normative; argument shapes are normative for required arguments and may be extended.

### `identity`

Returns the Sidecar's own identity.

```
input:  {}
output: {
  shadowname:  string,
  pk:          string,                  // multibase Ed25519 of the Subject's signing key
  credentials: [
    { kind:      "org_affiliation",     // v0.2 defines only this kind (RFC 0001 §6.2)
      issuer:    string,
      org:       string,                 // REQUIRED for org_affiliation
      expiresAt: string                  // ISO 8601
    }
  ]
}
```

### `resolve`

Resolves a Shadowname via Shadownet RFC 0001 §5 (DNS lookup + AgentCard fetch) and returns what it found, **without** adding to the contact graph.

```
input:  { name: string }
output: {
  shadowname: string,
  pk:         string,
  endpoint:   string                     // A2A URL
}
error:  "resolve_failed" | "unreachable"
```

### `contacts`

Lists known contacts.

```
input:  { query?: string }               // substring match on shadowname or displayName
output: { contacts: [
  { shadowname:  string,
    displayName?: string,
    grants:      string[],
    lastSeen?:   string                  // ISO 8601
  }
]}
```

### `contact_detail`

Full record for one contact.

```
input:  { name: string }
output: {
  shadowname:  string,
  displayName?: string,
  pk:          string,                   // cached from AgentCard
  endpoint:    string,                   // cached from AgentCard
  grants:      string[],
  credentials: [                          // credentials the peer has presented and we have cached
    { kind: string, issuer: string, expiresAt: string }
  ],
  profile?:    ContactProfile,            // see §6
  addedAt:     string,                    // ISO 8601
  lastSeen?:   string
}
error:  "not_contact"
```

### `add_contact`

Adds a Shadowname to the contact graph. If the Sidecar holds stranger-review items from this Shadowname, it MUST graduate them to inbox after the contact is added.

```
input:  {
  name:         string,
  displayName?: string,
  grants?:      string[],                // defaults to ["messaging"]
  profile?:     ContactProfile           // optional initial profile; see §6
}
output: {
  shadowname:    string,
  trustWarning?: {                       // present iff the contact's cached credentials reference issuers not in the Subject's trust store
    untrustedIssuers: string[]           // issuer domains the host LLM may want to surface to the user
  }
}
error:  "resolve_failed" | "already_contact"
```

**Empty trust store note.** Per Shadownet RFC 0001 §7.1, Sidecars ship with an empty trust store. When the contact's AgentCard or first-contact envelope references an `iss` not yet in the trust store, the Sidecar MUST still permit the add (the contact graph and trust store are independent surfaces) but SHOULD surface the issuer mismatch via `trustWarning` so the host LLM can prompt the user to consider trusting that issuer. Trust-store edits themselves are not exposed through this MCP surface (§9).

### `grant`

Grants or revokes per-contact permissions.

```
input:  { name: string, grant: string, allowed: boolean }
output: { ok: true }
error:  "not_contact" | "unknown_grant"
```

v0.2 defines one grant: `messaging` (the contact's envelopes reach `inbox`; clearing it routes future inbound to `stranger_review` per Shadownet RFC 0001 §9). Future revisions MAY add scoped grants.

### `set_contact_profile`

Updates the local-only profile on a contact. See §6 for the ContactProfile shape. Passing `{}` clears all fields. Partial updates are not defined; clients SHOULD read the current profile via `contact_detail` and submit the full desired state.

```
input:  { name: string, profile: ContactProfile }
output: { ok: true }
error:  "not_contact"
```

### `send`

Sends a Shadownet-enveloped message.

```
input:  {
  to:         string,                    // recipient Shadowname
  body: {                                 // per RFC 0001 §8.5
    text?:    string,
    intent?:  string,                    // OPTIONAL URI; e.g. "urn:shadownet:intent:coordinate_v1" (§5)
    data?:    object                     // per intent schema if intent is known
  },
  contextId?: string                     // A2A contextId; new context if absent
}
output: {
  messageId: string,                     // the envelope's `id`
  contextId: string,
  status:    "accepted" | "rejected",
  error?:    string                       // Shadownet error code (URN suffix); present iff status=="rejected"
}
```

**Reply auto-add (informational).** When the recipient replies on the same `contextId`, per Shadownet RFC 0001 §9 the recipient's Sidecar auto-adds the Subject to its contact graph (the binding `contextId` ↔ outbound proves the reply is genuine). Symmetrically: when the recipient's reply arrives back at this Sidecar, it is treated as inbound from a known contact even if no prior `add_contact` was issued — host LLMs do not need to call `add_contact` for reply auto-onboarding.

### `respond`

Responds within an existing thread (same `contextId`). Equivalent to `send` with the `contextId` filled in from a prior inbound message; provided as a separate tool to make the host LLM's reply intent explicit and idempotency-friendly.

```
input:  { contextId: string, body: { text?, intent?, data? } }
output: { messageId, status, error? }
```

### `coordinate`

Initiates a coordination flow with a contact. The peer's Shadow may negotiate autonomously — the Subject is not interrupted until both agents reach agreement.

```
input:  {
  name:     string,                       // recipient Shadowname
  activity: string,                       // e.g. "coffee", "dinner", "meeting"
  details?: string                        // constraints — e.g. "Thursday morning", "downtown"
}
output: { messageId, contextId }
```

The Sidecar MUST construct an envelope with `body.intent = "urn:shadownet:intent:coordinate_v1"` and `body.data` per §5.1, then dispatch via `send`. The peer's response arrives asynchronously as an `inbox.message` event.

After calling this, the host LLM SHOULD end its turn and await the inbound event — it MUST NOT poll `inbox` in a loop.

### `confirm_plan`

Confirms an agreed coordination plan and sends a `confirm_plan_v1` envelope to the peer. Called after the Subject approves a proposed plan (the response from a `coordinate` flow).

```
input:  {
  name:      string,                     // recipient Shadowname
  contextId: string,                     // thread to confirm within
  plan:      PlanObject                  // §5.0
}
output: { messageId }
```

### `accept_plan`

Accepts a coordination plan that was proposed by a peer (an inbound `confirm_plan_v1` envelope). Called after the Subject says "yes" to a proposal.

```
input:  {
  name:             string,              // recipient Shadowname
  contextId:        string,
  acceptsMessageId: string                // the messageId of the peer's confirm_plan envelope
}
output: { messageId }
```

After this returns, the coordination is complete on both sides.

### `inbox`

Lists pending inbound messages with full body content.

```
input:  {
  since?:         string,                // opaque cursor; null = beginning of retention window
  contact?:       string,                // filter to envelopes from this Shadowname
  intent?:        string,                // filter to envelopes matching this intent URI
  includeReview?: boolean,               // default false; when true, includes stranger_review items
  limit?:         integer                // default 50
}
output: {
  items: [
    { messageId:  string,
      contextId:  string,
      from:       string,                // sender Shadowname
      receivedAt: string,                // ISO 8601
      status:     "inbox" | "stranger_review",
      body: {
        text?:    string,
        intent?:  string,
        data?:    object
      }
    }
  ],
  nextSince: string | null
}
```

The cursor `since` is opaque to the host LLM — it MUST NOT parse or compare it ordinally. The only normative operation is to echo `nextSince` from one call as `since` in the next.

### `inbox_wait`

Long-polls for inbox events. Holds the call open until events arrive or the timeout elapses. Suitable for host LLMs whose MCP SDK does not dispatch the `notifications/shadownet/*` namespace, and the RECOMMENDED default inbound delivery mechanism for all deployments.

```
input:  {
  timeout_seconds?: integer,             // default 30; server clamps to ≤ 90
  last_event_id?:   string | null        // default null = "deliver events from now on"
}
output: {
  events:        [ { eventId, event, occurredAt, data } ],
  next_event_id: string | null
}
```

`event` and `data` shapes MUST match the corresponding entry in §7. `eventId` is **opaque** to clients — host LLMs MUST NOT parse it. The only normative operation is to pass the most recently received value back as `last_event_id` on the next call.

Cross-transport dedupe: the same event delivered via the MCP notification path (§7 Path 1) and `inbox_wait` MUST carry byte-identical `eventId` strings. Host LLMs receiving the same event on both transports dedupe by string equality on `eventId`.

#### Server behavior

- If events newer than `last_event_id` are queued, return them immediately with `next_event_id` set to the most recent delivered id.
- Otherwise, park the call on a per-tenant condition for up to `min(timeout_seconds, 90)` seconds; wake on new events or timeout.
- On timeout with no new events, return `{ events: [], next_event_id: <current high-water mark> }`. If the tenant has never had an event, `next_event_id` is `null`.
- Sidecars SHOULD retain enough history to support cursor resumption across short disconnects (RECOMMENDED: at least 5 minutes or 100 events per tenant).

#### Client behavior

- The host LLM SHOULD run this loop in a background worker — not from the LLM reasoning loop. The tool is transport, not deliberation.
- The host LLM SHOULD mark this tool `disable-model-invocation` (or the host's equivalent) so the LLM does not invoke it.
- The host LLM MUST re-invoke immediately after each successful return (no inter-call sleep on success). The server-side timeout clamp provides the natural pacing.
- On transport error: exponential backoff RECOMMENDED, starting ~1 s, capped ~30 s, with up to 25% jitter.

## 5. Intent Profiles

This section defines the three intent URIs used by the v0.2 coordination flow. Each intent specifies the `body.data` shape for an envelope carrying that intent. Sidecars consuming these intents MUST surface the body content to the host LLM unchanged; Sidecars MAY apply schema validation and reject malformed `data` with a `payload_invalid` error.

Intents are URIs in the `urn:shadownet:intent:<name>_v<major>` namespace. Each intent is independently versioned; a v2 intent uses a new URI (`..._v2`). Implementations MAY define additional intents under their own URI namespace; receivers treat unknown intents per Shadownet RFC 0001 §8.5 (surface as opaque, do not reject).

### 5.0 Shared types

**`PlanObject`** describes a concrete agreed plan:

```
PlanObject = {
  activity:     string,                  // e.g. "dinner", "meeting", "coffee"
  when:         string,                  // ISO 8601 datetime or interval
  where: {
    city?:    string,
    type?:    string,                    // e.g. "park", "restaurant", "office", "venue"
    address?: string,
    name?:    string,                    // venue name
    geo?:     { lat: number, lon: number }
  },
  participants: string[],                 // shadownames, including all parties
  notes?:       string                    // free-form for human display
}
```

### 5.1 `urn:shadownet:intent:coordinate_v1`

Propose an activity with constraints. Used by the `coordinate` tool.

```
data: {
  activity: string,                       // e.g. "dinner", "coffee"
  details?: string                        // free-form constraints
}
```

A `coordinate_v1` envelope opens a coordination conversation. The receiver's expected reply is a `confirm_plan_v1` (when ready to commit) or a free-form `respond` (when more negotiation is needed).

### 5.2 `urn:shadownet:intent:confirm_plan_v1`

Confirm a specific agreed plan. Used by the `confirm_plan` tool.

```
data: PlanObject                          // see §5.0
```

A `confirm_plan_v1` envelope says "this is the agreed plan; please accept it." The receiver responds with `accept_plan_v1` or a free-form `respond`.

### 5.3 `urn:shadownet:intent:accept_plan_v1`

Accept a peer's confirmed plan. Used by the `accept_plan` tool.

```
data: {
  acceptsMessageId: string                // messageId of the confirm_plan envelope
}
```

An `accept_plan_v1` envelope is the canonical terminal — both sides now consider the plan committed. Implementations SHOULD use the `accept_plan_v1`'s receipt as the trigger to write the plan into the user's calendar / task system.

## 6. ContactProfile

A ContactProfile is local-only metadata the Subject (or the host LLM at the Subject's direction) attaches to a contact. Its purpose is to give the host LLM's reasoning loop context about how the Subject thinks of this contact — analogous to the notes a person keeps next to a name in an address book, scaled up so an LLM can use them across hundreds of contacts.

A ContactProfile MUST NOT be transmitted to the contact or to any other peer. It MUST NOT appear in any over-the-wire artifact (envelope, AgentCard, credential, status list).

### Shape

```json
{
  "notes":     "Contractor working with Bob on Project Foo. Respected. Prioritize his messages.",
  "priority":  "high",
  "tags":      ["project_foo", "external"],
  "expiresAt": "2026-08-01T00:00:00Z"
}
```

All fields OPTIONAL:

| Field | Type | Purpose |
| --- | --- | --- |
| `notes` | string | Free-form text the Subject wrote about the contact. The host LLM SHOULD surface this to its reasoning loop as context when the contact is involved in an interaction. Maximum 4 KiB. |
| `priority` | enum: `low` \| `normal` \| `high` | Routing hint: how urgently to surface inbound from this contact. Default `normal`. |
| `tags` | string array | Categories the Subject has applied (project codenames, relationship type, affiliation, etc.). |
| `expiresAt` | RFC 3339 timestamp | Auto-archive date for time-bounded relationships. Sidecars MAY surface a reminder when an expiring contact's date approaches. |

Sidecars MUST persist the profile, surface it via `contact_detail`, and never include it in over-the-wire artifacts.

## 7. Inbound notifications

Two delivery paths are defined. A Sidecar MUST support at least one:

1. **MCP notifications** — server-initiated push on the existing MCP channel (Path 1, below).
2. **Long-poll** — host-LLM-driven via `inbox_wait` (Path 2).

Host LLMs that subscribe to neither MUST fall back to one-shot polling of `inbox`.

Path 2 (`inbox_wait`) is RECOMMENDED as the default. Pick based on deployment:

- Path 1 fits when the host LLM's MCP stack dispatches arbitrary notification namespaces and the MCP transport keeps a push channel open reliably. Lowest infrastructure cost when it works.
- Path 2 fits hosts that cannot keep a server-push channel open (laptops behind NAT, idle-killing middleboxes), MCP clients that only support request/response, and deployments that want the pull model for auditing, rate-control, or batching.

A Sidecar MAY implement both. Receivers consuming both paths dedupe by `eventId`.

### Path 1: MCP server-initiated notification

Sidecars that implement MCP server-initiated notifications SHOULD push in the `notifications/shadownet/` namespace on new inbound activity:

- `notifications/shadownet/inbox.message`
- `notifications/shadownet/task.update`

Notification params mirror the corresponding `data` shape in §7 Events plus an `eventId` field that MUST be byte-identical to the `eventId` the same event would carry via `inbox_wait`.

### Path 2: Long-poll via `inbox_wait`

See `inbox_wait` (§4) for the full tool contract. This is the RECOMMENDED default for all deployments.

### Events

| `event` | `data` shape | When |
| --- | --- | --- |
| `inbox.message` | `{ messageId, contextId, from, intent?, status }` | New A2A envelope accepted into `inbox` or `stranger_review`. Body is fetched via `inbox`. |
| `task.update` | `{ contextId, taskId, status }` | An A2A task changed status (e.g., the peer's `task:get` returned a new state). Only emitted when application logic has opened an A2A Task; Shadownet's own envelope responses are A2A `Message`s and do not generate this event. |

Future events are added by name; v0.2 host LLMs MUST ignore unrecognised event types rather than failing.

## 8. Behavioural requirements

- Tools MUST be idempotent where the operation is naturally idempotent (`grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage (subject to user-controlled retention).
- The Sidecar MUST NOT silently expand the Subject's exposed data beyond what each tool's input names.

## 9. Out of scope

- **Trust-store editing.** Adding or removing trusted issuers MUST NOT be reachable from the host LLM's tool surface — prompt injection could expand trust. Belongs in the Sidecar's account portal.
- **Audit endpoints.** Sidecars MAY expose audit logs over a separate surface; format is not standardized.
- **OAuth-style scoped tokens.** Bearer tokens are opaque and grant full access; deployments needing scopes issue separate tokens per scope.