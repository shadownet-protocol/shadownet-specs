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

Adds a resolved entity to the contact graph. Optionally accepts a local-only profile that the Subject (or the host agent at the Subject's direction) attaches to the contact — see [§ Contact profile](#contact-profile).

```
input:  {
  shadowname:  string,
  displayName?: string,
  grants?:     string[],                    ; subset of recognized grant strings (§ social_grant)
  profile?:    ContactProfile               ; local-only metadata; NEVER transmitted
}
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

Long-polls for inbox events. Holds the call open until events arrive or the timeout elapses. Suitable for host agents that cannot host an inbound HTTPS webhook (laptops, MCP-only runtimes) and for hosts whose MCP SDK does not dispatch the [`notifications/shadownet/*`](#path-1-mcp-server-initiated-notification-in-band) namespace.

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

Cross-transport dedupe: the same event delivered via the webhook path ([§ Path 2](#path-2-webhook-out-of-band)) and `social_inbox_wait` MUST carry byte-identical `event_id` strings. Host agents receiving the same event on both transports dedupe by string equality on `event_id`.

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

#### Coexistence with webhooks

A tenant MAY have both a webhook subscriber and an active `social_inbox_wait` consumer. The Sidecar delivers each event to both channels; receivers dedupe by `event_id`. Most host agents will choose one path — plugins SHOULD make the two mutually exclusive in their default configuration to avoid double processing.

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

v0.1 defines two grant strings:

| Grant | Permits |
| --- | --- |
| `messaging` | The contact may send messages at all. Required for any inbound from this contact to reach `social_inbox`. |
| `coordinate` | The contact may initiate negotiations that require structured coordination (calendar, payments, group flows once defined). Implies `messaging`. |

Future RFCs MAY add scoped grants. Verbs that resemble "this contact's introduction of strangers is sufficient" (web-of-trust vouching) are deliberately not defined at v0.1: a leaked grant of that shape becomes a phishing vector, and the gain over surfacing "this contact introduced them" as a UI hint in the quarantine surface is small.

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

### `social_quarantine_list`

Lists pending quarantined inbound — items the receiver has held because the sender is not a known contact (per [RFC-0006 §Routing and quarantine](./0006-a2a-profile.md#routing-and-quarantine)). Items returned by this tool MUST NOT have triggered host-agent invocation; their summaries are sender-supplied, not LLM-derived.

```
input:  { since?: timestamp, limit?: number }
output: {
  items: [{
    quarantineId:    string,
    senderShadowname: string,
    senderDid:       string,
    purpose:         string | null,       ; from hints.purpose, e.g. "invitation"
    summary:         string,              ; sender-supplied; typically payload.text
    affiliation:     string | null,       ; sender's affiliation DID if presented, else null
    introducer:      string | null,       ; sender-claimed introducer DID; UI hint only
    flags:           string[],            ; e.g. ["rate-limited", "suspected-spam"]
    receivedAt:      timestamp
  }]
}
```

`flags` is populated by the gateway's rules-only analysis. Recognized values at v0.1: `"rate-limited"`, `"suspected-spam"`, `"unknown-affiliation-issuer"`. Implementations MAY add additional flag strings; host agents MUST tolerate unrecognized flags and SHOULD surface them as-is.

### `social_quarantine_review`

Reviews a quarantined item — accepts (adding the sender to the contact graph) or rejects. Reject + block prevents future inbound from the same sender DID without surfacing it again.

```
input: {
  quarantineId: string,
  decision:     "accept" | "reject" | "reject_and_block",
  ; on accept:
  displayName?: string,
  grants?:      string[],                  ; defaults to ["messaging"]
  profile?:     ContactProfile             ; local-only; see § Contact profile
}
output: {
  contactId?: string,                       ; present iff decision = "accept"
  ok: true
}
```

On `accept`, the receiver MUST (a) add the sender to the contact graph with the named grants, (b) transition the corresponding A2A task to a state the sender's Shadow can observe as accepted, and (c) deliver the original quarantined payload to `social_inbox` so subsequent host-agent processing can proceed normally.

On `reject` or `reject_and_block`, the receiver MUST transition the sender's task to `failed` with reason `peer_declined`. The sender learns the request was not accepted but receives no detail about why. `reject_and_block` additionally records the sender DID in a local block list so future inbound from the same DID is dropped at the gateway before quarantine.

## Contact profile

A **ContactProfile** is local-only metadata the Subject (or the host agent at the Subject's direction) attaches to a contact. Its purpose is to give the host agent's reasoning loop context about how the Subject thinks of this contact — analogous to the notes a person keeps next to a name in an address book, scaled up so an LLM can use them.

A ContactProfile MUST NOT be transmitted to the contact or to any other peer. It is stored alongside the contact graph entry and surfaced back to the host agent on inbound and outbound that involves this contact (typically via `social_contact_detail`).

### Shape

```json
{
  "notes":          "Contractor working with Bob on Project Foo. Respected. Prioritize his messages.",
  "priority":       "high",
  "collaborate_on": ["Project Foo", "Y", "Z"],
  "expires_at":     "2026-08-01T00:00:00Z"
}
```

Fields (all OPTIONAL):

| Field | Type | Purpose |
| --- | --- | --- |
| `notes` | string | Free-form text the Subject wrote about the contact. The host agent SHOULD surface this to its reasoning loop as context. Maximum 4 KiB. |
| `priority` | enum: `low` \| `normal` \| `high` | Routing hint for the host agent: how urgently to surface inbound. Default `normal`. |
| `collaborate_on` | string array | Topics or projects the Subject has scoped the relationship to. Host agents MAY use these to filter or label inbound. |
| `expires_at` | RFC 3339 timestamp | Optional auto-archive date. Suitable for contractor relationships and time-bounded collaborations. Sidecars MAY surface a reminder when an expiring contact's date approaches. |

Sidecars MUST persist the profile, surface it via `social_contact_detail`, and never include it in over-the-wire artifacts. Sidecars SHOULD provide an MCP tool to update the profile post-creation; the canonical shape is `social_set_contact_profile` with the same `profile` field as `social_add_contact`.

### `social_set_contact_profile`

Updates the local-only profile on an existing contact.

```
input:  { contactId: string, profile: ContactProfile }
output: { ok: true }
```

A `profile` of `{}` clears all fields. Partial updates are not defined at v0.1; clients SHOULD read the current profile via `social_contact_detail` and submit the full desired state.

## Optional tools

`social_present` — explicitly trigger a credential presentation to a peer (for testing or unusual flows). Most host agents will never need it; the Sidecar handles presentations automatically during the A2A handshake.

`social_audit` — returns a structured audit log of host-agent actions. Strongly recommended; not strictly required at v0.1.

## Behavioural requirements

- Tools MUST be idempotent where the operation is naturally idempotent (`social_grant` with the same input yields the same state).
- The Sidecar MUST log every tool call to local storage (subject to user-controlled retention).
- The Sidecar MUST NOT silently expand the Subject's exposed data beyond what each tool's input names.
- **Cost guarantee.** Inbound from a sender that is not a known contact with the `messaging` grant MUST NOT appear in `social_inbox`, MUST NOT trigger any `notifications/shadownet/*` MCP notification beyond optional out-of-band quarantine alerts, and MUST NOT cause the host agent's reasoning loop to be invoked. Such inbound is surfaced exclusively through `social_quarantine_list` until the Subject explicitly reviews it. This is the local enforcement of the [RFC-0006 §Cost guarantee](./0006-a2a-profile.md#cost-guarantee).
- **Contact profile is local-only.** A `ContactProfile` MUST NOT appear in any over-the-wire artifact (A2A envelope, VP, SNS record, integration bundle, OAuth token, webhook payload). Implementations that synchronize state across multiple Sidecars for the same Subject MAY include profile data in that synchronization channel provided it remains under the Subject's exclusive control.

## Inbound notifications

Three delivery paths are defined. A Sidecar MUST support at least one:

1. **MCP notifications** — server-initiated push on the existing MCP channel (Path 1, below).
2. **Webhook** — out-of-band HTTPS push to a host-agent-controlled URL (Path 2).
3. **Long-poll** — host-agent-driven via [`social_inbox_wait`](#social_inbox_wait) (Path 3).

Host agents that subscribe to none of the above MUST fall back to one-shot polling of [`social_inbox`](#social_inbox).

All three paths are first-class and permanent. Pick based on deployment:

- Path 1 fits when the host agent's MCP stack dispatches arbitrary notification namespaces and the MCP transport keeps a push channel open reliably. Lowest infrastructure cost when it works.
- Path 2 fits cloud-to-cloud deployments where the host agent runs a reachable HTTPS endpoint. Familiar webhook ergonomics, replay defense, and retry semantics.
- Path 3 fits hosts that cannot keep a server-push channel open (laptops behind NAT, idle-killing middleboxes), MCP clients that only support request/response, and deployments that want the pull model for auditing, rate-control, or batching.

A Sidecar MAY implement more than one. Receivers consuming multiple paths dedupe by `event_id` (see [§ `social_inbox_wait`](#social_inbox_wait) and the Path 2 wire shape).

### Path 1: MCP server-initiated notification (in-band)

Sidecars that implement [MCP server-initiated notifications](https://modelcontextprotocol.io/) SHOULD push notifications in the `notifications/shadownet/` namespace on new inbound activity — one method per event name from [§ Events](#events):

- `notifications/shadownet/inbox.message`
- `notifications/shadownet/task.update`
- `notifications/shadownet/freshness.expired`
- `notifications/shadownet/presentation.failed`
- `notifications/shadownet/quarantine.pending`

Notification params mirror the corresponding webhook `data` shape (below) plus an `event_id` field that MUST be byte-identical to the `event_id` the same event would carry via [`social_inbox_wait`](#social_inbox_wait) or the webhook path. This is how host agents that consume more than one transport dedupe.

Sidecars emitting these notifications SHOULD advertise the `mcp-notifications` capability in their [integration bundle](./0008-onboarding.md#capability-flags). Host agents are responsible for confirming their MCP SDK dispatches these notification methods; SDKs that validate against a closed notification union will silently drop them, in which case the host agent SHOULD use Path 3 instead.

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
  "event_id":    "01HQZX...",
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

`event_id` is opaque to receivers and MUST be byte-identical to the `event_id` the same event would carry via [`social_inbox_wait`](#social_inbox_wait) or a Path 1 notification. Receivers dedupe across transports by string equality on this field.

The webhook payload deliberately does NOT carry the message *content* — the host agent is expected to call `social_inbox` for the actual payload. This keeps the webhook small, replay-safe, and avoids leaking content into webhook logs.

#### Receiver requirements

The host agent MUST:

1. Verify the HMAC. Compare `X-Shadownet-Sidecar-Sig` (after stripping the `sha256=` prefix) against `HMAC-SHA256(secret, body)`.
2. Reject deliveries whose `X-Shadownet-Sidecar-Ts` differs from local time by more than 5 minutes (replay defense).
3. Respond `2xx` on accepted delivery; any other status (including `5xx`) triggers retry.
4. Be idempotent on `event_id` (a delivery may arrive more than once due to retries, and the same event MAY also arrive via another path).

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
| `quarantine.pending` | `{ quarantineId, senderDid, purpose }` | New item placed in quarantine for the Subject's review. Carries no content; host agent retrieves details via `social_quarantine_list`. |

Future events are added by name; v0.1 host agents MUST ignore unrecognised event types rather than failing.

#### Compatibility headers

Sidecars MAY emit additional compatibility headers that carry the same HMAC-SHA256 in alternate formats demanded by other webhook ecosystems. When such headers are emitted:

- The canonical `X-Shadownet-Sidecar-Sig`, `-Ts`, and `-Id` headers MUST still be emitted.
- Receivers validating only a compatibility header MUST also verify `X-Shadownet-Sidecar-Ts` is within ±5 minutes of local time, OR explicitly accept the loss of replay defense (e.g., behind a documented config flag).
- Sender configuration enabling compatibility headers SHOULD log a one-line warning at startup naming the safety property being bypassed.

A widely-supported example is `X-Webhook-Signature: <hex HMAC-SHA256 of body, key=secret>` — raw hex, no prefix — used by [Hermes Agent webhooks](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/webhooks) and similar generic-HMAC adapters. The spec endorses no specific compatibility header; the pattern is what's normative.

## Open questions

- Whether to define a tool for adjusting the trust store (`social_trust_add`, `social_trust_remove`) or leave that to a separate config/UI surface.
- Whether `social_send` should accept multi-recipient input directly (group fan-out at the Sidecar) or require the host agent to call N times.
- Whether webhook secrets should be rotatable without re-registration.
