---
rfc: 0002
title: Shadownet MCP Control Surface
version: shadow1
shadownet: urn:shadow:v1
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet MCP Control Surface

## 1. Introduction

This document defines the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) tool surface a Shadownet Sidecar exposes to its host LLM. It is the local control plane between an MCP-capable host (Claude Desktop, Hermes, OpenClaw, any MCP client) and a Sidecar that holds the Subject's keys, contact graph, and message state.

This is companion to the [Shadownet Protocol](./0001-shadownet.md) (RFC 0001, `urn:shadow:v1`). The wire defined there — Sidecar ↔ Sidecar over A2A — is unchanged by this document. Shadownet wire conformance does not require implementing this companion; deployments with proprietary host integration are free to ignore it. However, the tool surface defined here is intended to be the canonical drop-in surface for any MCP-capable host LLM, so that host integrations work across Sidecar implementations without per-implementation glue.

In A2A's own framing (`a2a-and-mcp.md`), MCP is the *agent ↔ tools* layer while A2A is the *agent ↔ agent* layer. This document defines the tools side: the Sidecar exposes a small set of MCP tools so the host LLM can read its inbox, send envelopes, manage its contact graph, and drive coordination flows.

## 2. Transport and authentication

A Sidecar exposes an MCP server over [streamable HTTP](https://modelcontextprotocol.io/) at a Sidecar-configurable URL. The host LLM connects to that URL using any MCP-compliant client.

Every MCP request carries:

```
Authorization: Bearer <opaque-token>
```

The token is issued by the Sidecar's onboarding surface (see the onboarding companion). It is opaque to the host LLM: it identifies which Subject's Shadow the host is acting for, and grants full access to the tool surface defined below. Finer-grained authorization (per-tool scopes, time-limited tokens) is a Sidecar implementation choice and not standardized in v1.

A Sidecar MUST reject requests with missing or invalid tokens using MCP's standard error mechanism.

## 3. Conventions

Tool names use **snake_case** (MCP idiom). JSON argument and result field names use **camelCase** (Shadownet [RFC 0001 §2 naming table](./0001-shadownet.md#2-conventions)). Value strings (statuses, kinds, grant names, intent URI suffixes) use **snake_case**. Event names use dotted lowercase.

Shadowname strings appearing in arguments and results are canonical lowercase per Shadownet RFC 0001 §5.1. The token authenticates the Subject; no `subject` parameter is carried on individual calls.

Error responses use MCP's standard error format. When an error mirrors a Shadownet wire error (e.g., a `send` that the recipient rejected), the MCP error's identifier matches the Shadownet wire error code (e.g., `creds_rejected`).

## 4. Tools

A v1 Sidecar MUST expose the **REQUIRED** tools below. **RECOMMENDED** tools are convenience wrappers; hosts that wish to bypass them MAY call `send` directly with custom intent URIs.

### 4.1 Identity and discovery

#### `identity`  (REQUIRED)

Returns the Sidecar's own identity — useful for the host LLM at session start.

```
input:  {}
output: {
  shadowname:  string,
  pk:          string,                  // multibase Ed25519 of the Subject's signing key
  credentials: [
    { kind:      "personhood" | "org" | "org_affiliation",
      issuer:    string,                // issuer domain
      org?:      string,                // present iff kind == "org_affiliation"
      expiresAt: string                 // ISO 8601
    }
  ]
}
```

#### `resolve`  (REQUIRED)

Resolves a Shadowname via Shadownet RFC 0001 §5 (DNS lookup + AgentCard fetch). Returns what was found; does not change local state.

```
input:  { name: string }
output: {
  shadowname: string,
  pk:         string,                   // multibase Ed25519
  endpoint:   string                    // A2A URL
}
error:  "resolve_failed" | "unreachable"
```

### 4.2 Contact graph

#### `contacts`  (REQUIRED)

Lists known contacts. Abbreviated records, paginated — designed to scale to hundreds or thousands of contacts. For the full record of a single contact (including profile), call `contact_detail`.

```
input:  {
  query?: string,                        // substring match on shadowname or displayName
  since?: string,                        // opaque cursor
  limit?: integer                        // default 50; max 200
}
output: {
  contacts: [
    { shadowname:  string,
      displayName?: string,
      priority?:   "low" | "normal" | "high",   // from the local profile, surfaced for fast filtering
      tags?:       string[],                    // from the local profile
      lastSeen?:   string                        // ISO 8601 of last inbound or outbound envelope
    }
  ],
  nextSince: string | null               // pass back on next call; null when no further records
}
```

#### `contact_detail`  (REQUIRED)

Returns the full local record for a single contact.

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

#### `add_contact`  (REQUIRED)

Adds a Shadowname to the contact graph. If the Sidecar holds stranger-review items from this Shadowname, it MUST graduate them to inbox after the contact is added; the host LLM observes them via a subsequent `inbox` call.

```
input:  {
  name:         string,
  displayName?: string,
  grants?:      string[],                 // defaults to ["messaging"]
  profile?:     ContactProfile            // optional initial profile; see §6
}
output: { shadowname: string }
error:  "resolve_failed" | "already_contact"
```

#### `remove_contact`  (REQUIRED)

Removes a Shadowname from the contact graph. Held envelopes from this Shadowname are not deleted — they revert to `stranger_review` and follow the Sidecar's retention policy. Removing a contact does NOT prevent future inbound; for that, call `block` separately.

```
input:  { name: string }
output: { ok: true }
error:  "not_contact"
```

#### `grant`  (REQUIRED)

Sets or clears a per-contact grant.

```
input:  { name: string, grant: string, allowed: boolean }
output: { ok: true }
error:  "not_contact" | "unknown_grant"
```

v1 defined grants:

| Grant | Permits |
| --- | --- |
| `messaging` | This contact's envelopes reach `inbox`. Clearing this routes future inbound from this Shadowname to `stranger_review` per Shadownet RFC 0001 §9. |

Future grants are added by string. Sidecars MUST return `unknown_grant` for grant strings they do not implement.

#### `set_contact_profile`  (REQUIRED)

Updates the local-only profile on a contact. See §6 for the ContactProfile shape. Passing `{}` clears all fields. Partial updates are not defined at v1; clients SHOULD read the current profile via `contact_detail` and submit the full desired state.

```
input:  { name: string, profile: ContactProfile }
output: { ok: true }
error:  "not_contact"
```

### 4.3 Block list

Block management is independent of the contact graph: a Subject can block a Shadowname they were never a contact with.

#### `block`  (REQUIRED)

Adds a Shadowname to the block list. Future envelopes from this Shadowname are dropped at the Sidecar's wire boundary (before routing) and never reach `inbox` or `stranger_review`. If the Shadowname is currently a contact, `block` does NOT remove the contact entry — call `remove_contact` separately if desired.

```
input:  { name: string, reason?: string }
output: { ok: true }
```

`reason` is local-only metadata for the Subject's own audit; never transmitted.

#### `unblock`  (REQUIRED)

Removes a Shadowname from the block list.

```
input:  { name: string }
output: { ok: true }
error:  "not_blocked"
```

#### `blocked`  (REQUIRED)

Lists currently blocked Shadownames.

```
input:  { since?: string, limit?: integer }
output: { blocked: [
  { shadowname: string, blockedAt: string, reason?: string }
], nextSince: string | null }
```

### 4.4 Sending

#### `send`  (REQUIRED)

Dispatches a Shadownet envelope. The Sidecar resolves the recipient, constructs the envelope per RFC 0001 §8, signs with the Subject's key, attaches credentials per its caching policy, and POSTs to the recipient's A2A endpoint.

```
input:  {
  to:         string,                    // recipient Shadowname
  body: {
    text?:    string,
    intent?:  string,                    // URI per RFC 0001 §8.5
    data?:    object
  },
  contextId?: string                     // A2A contextId for threading; new contextId if absent
}
output: {
  messageId: string,                     // ULID/UUID; the envelope's `id`
  contextId: string,                     // echoed or newly generated
  status:    "accepted" | "rejected",
  error?:    string                      // Shadownet error code (URN suffix); present iff status=="rejected"
}
```

`status: "accepted"` means the receiver returned a successful A2A `Message` response. `status: "rejected"` means the receiver returned an error; the `error` field carries the Shadownet error identifier (e.g., `creds_rejected`, `policy`, `rate_limited`).

#### `coordinate`  (RECOMMENDED)

Convenience wrapper around `send` for initiating a coordination flow. Sends an envelope with `body.intent = "urn:shadow:intent:coordinate.v1"` and a `body.data` shape carrying the activity and constraints.

```
input:  {
  name:     string,                      // recipient Shadowname
  activity: string,                      // e.g. "coffee", "dinner", "meeting"
  details?: string                       // e.g. "Thursday morning", "downtown"
}
output: same shape as send (messageId, contextId, status, error?)
```

The envelope body the Sidecar constructs is:

```json
{
  "intent": "urn:shadow:intent:coordinate.v1",
  "data":   { "activity": "<activity>", "details": "<details>" }
}
```

After calling this, the host LLM SHOULD await the peer's response asynchronously via `inbox_wait` rather than polling `inbox` in a loop.

#### `confirm_plan`  (RECOMMENDED)

Convenience wrapper for confirming an agreed coordination plan. Sends an envelope with `body.intent = "urn:shadow:intent:confirm_plan.v1"`.

```
input:  {
  name:      string,                     // recipient Shadowname
  contextId: string,                     // thread to confirm within
  plan:      object                      // application-defined plan structure
}
output: same shape as send
```

The envelope body is:

```json
{
  "intent": "urn:shadow:intent:confirm_plan.v1",
  "data":   { "plan": { ... } }
}
```

#### `accept_plan`  (RECOMMENDED)

Convenience wrapper for accepting a peer's confirmed plan. Sends an envelope with `body.intent = "urn:shadow:intent:accept_plan.v1"`.

```
input:  {
  name:               string,            // recipient Shadowname
  contextId:          string,
  acceptsMessageId:   string              // the messageId of the peer's confirm_plan envelope
}
output: same shape as send
```

The envelope body is:

```json
{
  "intent": "urn:shadow:intent:accept_plan.v1",
  "data":   { "acceptsMessageId": "<message_id>" }
}
```

The schemas for `coordinate.v1`, `confirm_plan.v1`, and `accept_plan.v1` are sketched here at the minimum field level so the canonical flow is implementable in v1. A formal Intent Profiles companion will normalize the full schemas (acceptable activity vocabularies, plan structure for scheduling vs payment vs introduction, etc.) at a later cadence.

### 4.5 Inbox

#### `inbox`  (REQUIRED)

Fetches held messages with full body content.

```
input:  {
  since?:         string,                // opaque cursor; null = beginning of retention window
  contact?:       string,                // filter to envelopes from this Shadowname
  includeReview?: boolean                // default false; when true, includes stranger_review items
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
  nextSince: string | null               // pass back on next call; null when no further items
}
```

The cursor `since` is opaque to the host LLM — it MUST NOT parse or compare it ordinally. The only normative operation is to echo `nextSince` from one call as `since` in the next.

#### `inbox_wait`  (REQUIRED)

Long-polls for new inbox events. Returns immediately if events are queued; otherwise parks until events arrive or the timeout elapses.

```
input:  {
  since?:         string,
  timeout?:       integer,               // seconds; default 30; server clamps to ≤ 90
  includeReview?: boolean                // default false
}
output: {
  events: [
    { eventId:    string,                // opaque
      event:      "inbox.message",
      occurredAt: string,                // ISO 8601
      data: {
        messageId: string,
        contextId: string,
        from:      string,                // sender Shadowname
        intent?:   string,                // body.intent if present in the envelope
        status:    "inbox" | "stranger_review"
      }
    }
  ],
  nextSince: string | null
}
```

Events carry **metadata only** — no body. To read the actual envelope body (`text`, `intent` data, etc.), the host LLM calls `inbox` with the event's `messageId` cursor (or just calls `inbox(since: <cursor>)` to read all bodies since the cursor).

This two-step pattern (cheap notification, separate body fetch) keeps the long-poll channel light and lets the host LLM decide which messages to materialize fully into its reasoning context.

Host LLMs SHOULD re-invoke `inbox_wait` immediately on return; the server-side timeout provides the natural pacing. On MCP transport error, exponential backoff starting ~1 s, capped ~30 s, RECOMMENDED.

## 5. Events

The Sidecar emits one event type, delivered via `inbox_wait` returns.

### 5.1 `inbox.message`

A new envelope reached `inbox` or `stranger_review` for this Subject. Event data shape per §4.5. The host LLM fetches body content via `inbox`.

A Sidecar MAY additionally emit MCP server-initiated notifications under the namespace `notifications/shadow/inbox.message`, mirroring the same event metadata. This is OPTIONAL and not required for v1 conformance; host LLMs whose MCP client supports notifications MAY consume them. When a Sidecar emits on both channels, events carry the same `eventId` so host LLMs dedupe by event identity.

### 5.2 Cursor semantics

`eventId` strings are opaque. Host LLMs MUST NOT parse them. The only normative operation is to pass them back as `since` on the next `inbox_wait` call (or as `since` on `inbox` to fetch bodies since that cursor).

On `inbox_wait` timeout with no new events, the Sidecar returns `{ events: [], nextSince: <current high-water mark> }`. The high-water mark is the most recent eventId the Sidecar has issued for this Subject; passing it back is safe (it produces an empty list on the next call until something new arrives).

Sidecars SHOULD retain enough event history to support cursor resumption across short disconnects (RECOMMENDED: at least 5 minutes or 100 events per Subject). When a client presents a `since` older than the retained window, the Sidecar MUST return events from the oldest available point; clients detect the gap by noticing the returned events do not start where they expected.

## 6. ContactProfile

A ContactProfile is local-only metadata the Subject (or the host LLM at the Subject's direction) attaches to a contact. Its purpose is to give the host LLM's reasoning loop context about how the Subject thinks of this contact — analogous to the notes a person keeps next to a name in an address book, scaled up so an LLM can use them across hundreds of contacts.

A ContactProfile MUST NOT be transmitted to the contact or to any other peer. It MUST NOT appear in any over-the-wire artifact (envelope, AgentCard, credential, status list). Sidecars MAY synchronize profiles across multiple Sidecars run by the same Subject (e.g., desktop and phone for one user) under the Subject's exclusive control; that synchronization channel is out of scope here.

### Shape

```json
{
  "notes":     "Contractor working with Bob on Project Foo. Respected. Prioritize his messages.",
  "priority":  "high",
  "tags":      ["project-foo", "external"],
  "expiresAt": "2026-08-01T00:00:00Z"
}
```

All fields OPTIONAL:

| Field | Type | Purpose |
| --- | --- | --- |
| `notes` | string | Free-form text the Subject wrote about the contact. The host LLM SHOULD surface this to its reasoning loop as context when the contact is involved in an interaction. Maximum 4 KiB. |
| `priority` | enum: `low` \| `normal` \| `high` | Routing hint for the host LLM: how urgently to surface inbound from this contact. Default `normal`. |
| `tags` | string array | Categories the Subject has applied to the contact (project codenames, relationship type, organizational affiliation, etc.). Host LLMs MAY use these to filter, group, or color inbound. |
| `expiresAt` | RFC 3339 timestamp | Auto-archive date. Suitable for contractor relationships and time-bounded collaborations. Sidecars MAY surface a reminder to the host LLM when an expiring contact's date approaches. |

Sidecars MUST persist the profile, surface it via `contact_detail`, and never include it in over-the-wire artifacts.

## 7. Reliability and cost guidance (non-normative)

Host LLMs SHOULD run `inbox_wait` from a background worker, not from the LLM reasoning loop. The tool is transport, not deliberation; calling it from the reasoning loop wastes tokens and produces unpredictable latency.

The Sidecar applies Shadownet's receiver classification (RFC 0001 §9) before any envelope reaches this MCP surface. Stranger envelopes whose credentials fail the receiver's `fromStranger` policy are dropped at the wire; the host LLM never sees them. Stranger envelopes that pass credential checks but lack a contact-graph entry are placed in `stranger_review` and visible only when the host LLM opts in via `includeReview: true`.

Host LLMs that route stranger-review items through their reasoning loop are spending tokens on potentially unsolicited content; this is a deliberate choice the host makes, not something this spec or Shadownet RFC 0001 normatively forbids.

Coordination wrappers (`coordinate`, `confirm_plan`, `accept_plan`) are non-blocking: they return as soon as the Sidecar has dispatched the envelope. The peer's response arrives asynchronously as an `inbox.message` event. Host LLMs SHOULD await the inbound event rather than polling, and SHOULD end their reasoning turn after dispatching coordination so the user is not held in a synchronous wait state.

## 8. Out of scope

The following are deliberately not in this MCP surface:

- **Full Intent Profile schemas.** This document sketches the minimum field shapes for `coordinate.v1` / `confirm_plan.v1` / `accept_plan.v1`. A separate Intent Profiles companion will normalize complete schemas, vocabulary, and additional intent URIs (scheduling, intro, payment, etc.).
- **Audit endpoints.** Sidecars MAY expose audit logs over a separate surface; not standardized.
- **OAuth-style scoped tokens.** Bearer tokens are opaque and grant full access; deployments needing scoped tokens issue separate tokens per scope.
- **Trust-store editing tools.** Trust changes are infrequent and deliberate; they belong to the Sidecar's account portal, not the host LLM's tool surface.
- **ContactProfile synchronization between Sidecars.** When a Subject runs multiple Sidecars (desktop + phone), profile sync is the Sidecar implementation's concern, not this MCP surface.

## Appendix A — Example session

Host LLM connects, lists contacts, sees a new inbox event, fetches the body, responds with a coordination wrapper.

```
→ tool identity {}
← { shadowname: "alice@sh4dow.org",
    pk: "z6MkAlicePub...",
    credentials: [
      { kind: "personhood", issuer: "sca.sh4dow.org",
        expiresAt: "2026-08-29T00:00:00Z" }
    ]}

→ tool contacts { query: "bob" }
← { contacts: [
    { shadowname: "bob@example.org", displayName: "Bob",
      priority: "high", tags: ["project-foo"],
      lastSeen: "2026-05-29T08:15:00Z" }
  ], nextSince: null }

→ tool inbox_wait { timeout: 30 }
← { events: [
    { eventId: "evt_7c3f",
      event: "inbox.message",
      occurredAt: "2026-05-29T10:30:00Z",
      data: { messageId: "01HZ7K3CWAB4D6N5XT0M2EXAMPLE",
              contextId: "01HZ7K2BV5R2K0DW3FCONTEXT0001",
              from: "bob@example.org",
              intent: "urn:shadow:intent:coordinate.v1",
              status: "inbox" } }
  ], nextSince: "evt_7c3f" }

→ tool inbox { since: null, contact: "bob@example.org" }
← { items: [
    { messageId: "01HZ7K3CWAB4D6N5XT0M2EXAMPLE",
      contextId: "01HZ7K2BV5R2K0DW3FCONTEXT0001",
      from: "bob@example.org",
      receivedAt: "2026-05-29T10:30:00Z",
      status: "inbox",
      body: { text: "Want to grab dinner Thursday?",
              intent: "urn:shadow:intent:coordinate.v1",
              data: { activity: "dinner", details: "Thursday" } } }
  ], nextSince: "01HZ7K3CWAB4D6N5XT0M2EXAMPLE" }

# Host LLM decides to confirm a specific plan
→ tool confirm_plan {
    name: "bob@example.org",
    contextId: "01HZ7K2BV5R2K0DW3FCONTEXT0001",
    plan: { activity: "dinner", when: "2026-05-14T19:00:00Z", where: "Tiergarten" }
  }
← { messageId: "01HZ7K5XYZ...",
    contextId: "01HZ7K2BV5R2K0DW3FCONTEXT0001",
    status: "accepted" }
```

The host LLM then resumes `inbox_wait { since: "evt_7c3f" }` and ends its turn, awaiting Bob's `accept_plan` response.