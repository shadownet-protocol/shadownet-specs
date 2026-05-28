---
rfc: 0006
title: Shadownet Profile of A2A
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0006: Shadownet Profile of A2A

## Summary

Shadownet uses [Google's A2A protocol](https://google.github.io/A2A/) as its agent-to-agent transport. This RFC defines the Shadownet *profile* of A2A: required A2A features, the credential-presentation handshake, the message envelope, and async/offline patterns.

A2A provides delivery, streaming, task state, and auth hooks. It does **not** prescribe message content. Shadownet's profile keeps that property — message *content* is application-defined and will be standardized later as separate Interaction Profile RFCs (analogous to iCalendar over SMTP).

## Required A2A surface

A v0.1 Shadow MUST implement the A2A v1.0 methods:

| A2A method | Purpose in Shadownet |
| --- | --- |
| `message:send` | Single-shot message delivery. |
| `message:stream` | Streaming message delivery (SSE). |
| `task:get` | Status of a long-running negotiation. |
| `task:cancel` | Caller-initiated abort. |

A Shadow MUST publish an A2A agent card at `/.well-known/agent-card.json`. The card MUST list:

- `name` — display name
- `url` — A2A endpoint (matches SNS record's `endpoint`)
- `did` — the Shadow's DID (Shadownet extension)
- `publicKey` — JWK for the DID's signing key (Shadownet extension)
- `shadownet:v` — `"0.1"`

## Handshake

Every A2A request from one Shadow to another carries:

```
Authorization: Bearer <session-token-jwt>
X-Shadownet-Presentation: <verifiable-presentation-jwt>     ; on first request of a session
```

### Session token

A short-lived JWT (`exp - iat ≤ 300 s`) signed by the **caller's** DID:

```json
{
  "iss":   "did:key:z6MkCallerPubkey...",
  "aud":   "did:key:z6MkCalleePubkey...",
  "iat":   1759200000,
  "exp":   1759200300,
  "jti":   "<uuid>",
  "shadownet:v": "0.1",
  "purpose": "a2a-session"
}
```

`aud` MUST match the callee's DID as resolved via SNS. The callee MUST reject tokens whose `aud` is not its own DID.

### Verifiable Presentation

The `X-Shadownet-Presentation` header carries a [VP per RFC-0003 §Presentation](./0003-credentials.md#presentation). The presentation's `nonce`:

- On the caller's first request of a session, the caller picks a random `nonce` and includes it.
- The callee echoes that `nonce` in any subsequent challenge (see Re-presentation below).

The callee MUST validate the VP per [RFC-0004 §Trust evaluation](./0004-sca.md#trust-evaluation) before processing the request body.

### Re-presentation

If a request arrives with no `X-Shadownet-Presentation` and the callee has no cached presentation for the caller's DID within the freshness window, the callee MUST respond `401 Unauthorized` with:

```json
{ "error": "presentation_required", "nonce": "<verifier-issued nonce>" }
```

The caller then retries with a presentation bound to the supplied `nonce`.

## Message envelope (Shadownet extensions)

A2A messages carry `parts` of various `type`s. Shadownet adds one part type:

```json
{
  "type":      "shadownet/v1+envelope",
  "mediaType": "application/json",
  "data": {
    "shadownet:v":   "0.1",
    "intentId":      "urn:uuid:...",          ; correlates messages within a coordination
    "sessionId":     "urn:uuid:...",          ; correlates within an A2A task
    "interaction":   "urn:shadownet:int:scheduling.v1",  ; OPTIONAL; advisory hint
    "payload":       { ... }                  ; see below
  }
}
```

### Default form: free-form text

The default Shadownet payload is natural language. When `interaction` is absent, `payload` SHOULD be a JSON object containing at least a `text` field carrying the human-readable message, plus any structured hints the sender chooses to attach:

```json
{
  "shadownet:v": "0.1",
  "intentId":    "urn:uuid:...",
  "payload": {
    "text": "Hey — want to grab dinner Thursday or Friday next week?",
    "hints": { "deadline": "2026-05-15T17:00:00Z" }     ; optional, sender-defined
  }
}
```

Receivers MUST be able to handle this form. The receiving Shadow's host agent is expected to interpret `text` (typically with an LLM) and respond in kind. This is the form most v0.1 coordination is expected to take.

### Typed form: Interaction Profile

When both peers benefit from a machine-checkable schema (calendar slots, payment terms, structured forms), the sender MAY set `interaction` to a URI naming an **Interaction Profile** (defined in separate RFCs — e.g. scheduling, intro). In that case `payload` follows the schema set by that profile:

```json
{
  "shadownet:v": "0.1",
  "intentId":    "urn:uuid:...",
  "interaction": "urn:shadownet:int:scheduling.v1",
  "payload":     { ... profile-defined ... }
}
```

Typed profiles are an **opt-in optimization** for cases where structure prevents ambiguity (timezones, currencies, identifiers). They are not the default and the protocol does not require any particular profile to be supported.

### Verifier obligations

- Verifiers MUST accept envelopes with no `interaction` and surface the `payload` (typically `text`) to the host agent.
- Verifiers MUST NOT reject envelopes solely because `interaction` names an unknown profile; they MUST surface the `payload` to the host agent as opaque, and MAY include the unrecognized `interaction` URI as a hint.
- Verifiers MAY reject envelopes whose `payload` fails validation against a *known* `interaction` profile they have chosen to enforce.

## Routing and quarantine

A Shadownet receiver MUST classify every inbound A2A request — after [Handshake](#handshake) validation succeeds — into one of three routes before any further processing. This classification is the receiver's primary defense against unsolicited token consumption: only the **inbox** route is permitted to invoke the host agent's LLM.

### Cost guarantee

The host agent's LLM (or any equivalently expensive reasoning loop) MUST NOT be invoked by inbound traffic from a sender that is not a known contact with a `messaging` grant, except via the explicit user action of reviewing a quarantined item. Routing and quarantine decisions MUST be made by rules-only logic at the receiver (the Sidecar or its gateway).

A receiver MUST NOT use the host agent to interpret, score, summarize, or reply to traffic on the quarantine path. Summaries presented to the user in the quarantine UI SHOULD be drawn from sender-supplied fields (e.g. `payload.text`, `hints.purpose`), not from receiver-side LLM processing of those fields.

### Normative routing defaults

| Sender classification | Default route | Configurable to |
| --- | --- | --- |
| VP invalid, no personhood/affiliation floor met | **drop** (401) | — |
| Known contact, has `messaging` grant | **inbox** | — |
| Known contact, no `messaging` grant | **403, log** | inbox (per-contact opt-in) |
| Unknown sender, same affiliation as recipient | **inbox** (enterprise default) | quarantine (`compliance_mode: "always_quarantine"`) |
| Unknown sender, different or no affiliation | **quarantine** | inbox (personal opt-in; not RECOMMENDED) |

For personal Shadows (no recipient affiliation), the default for all unknown senders is **quarantine**. This is the safer default and matches the cost guarantee above.

For enterprise Shadows (recipient holds an AffiliationCredential), unknown senders bearing a valid AffiliationCredential whose `credentialSubject.affiliation` matches the recipient's are routed to **inbox** by default. This is the internal-mail analog: messages from `@acme.com` to `@acme.com` skip quarantine. Operators MAY override per-policy.

Cross-affiliation traffic (Acme to Google) is treated as unknown-sender unless the contact graph says otherwise.

### Quarantine surface

Quarantined items are held by the receiver (commonly the gateway, per [RFC-0005 §Gateway pattern](./0005-sns.md#gateway-pattern)) and surfaced to the Subject through:

- The MCP tools [`social_quarantine_list`](./0007-mcp-tools.md#social_quarantine_list) and [`social_quarantine_review`](./0007-mcp-tools.md#social_quarantine_review).
- Out-of-band notification (email, push) when the user has registered one. The notification itself MUST NOT contain message content beyond the sender Shadowname and a sender-supplied summary line; the user must open the quarantine surface to see the body.

The receiver MUST hold quarantined items for at least 14 days before expiry; SHOULD hold longer (default 30 days). The receiver MAY apply rate limits and abuse filtering before quarantining — items above the abuse threshold MAY be **flagged** in the quarantine surface but MUST NOT be silently discarded. The user retains the final discard choice.

### Sender behavior on quarantine

When a sender's request is quarantined, the receiver SHOULD respond with `202 Accepted` and an [A2A task](#async-and-offline) in `submitted` state. The sender's Shadow learns the message is awaiting recipient review but receives no information about who/why. The receiver MUST NOT leak the quarantine decision through error codes or timing.

When the recipient eventually approves or rejects, the receiver MAY transition the task to `working` (and deliver subsequent responses normally) or `failed` (with no detail beyond `peer_declined`). Senders MUST treat indefinite `submitted` state as "no response expected" and SHOULD give up after a documented timeout.

### Invitation envelopes

The first message from an unknown sender is typically an **invitation** — a request to be added to the recipient's contact graph. v0.1 does not define a typed Interaction Profile for invitations (Interaction Profiles are out of v0.1 scope); invitations are carried in the default free-form envelope with a recognized hint:

```json
{
  "shadownet:v": "0.1",
  "intentId": "urn:uuid:...",
  "payload": {
    "text": "Hi, I'm Alex. I'm a contractor working with Bob on Project Foo. I'd like to coordinate on Y and Z over the next two weeks.",
    "hints": {
      "purpose": "invitation",
      "proposed_collaboration": "Project Foo",
      "introducer_contact": "did:key:z6MkBobPubkey..."
    }
  }
}
```

Recognized `hints.purpose` values at v0.1: `"invitation"`. Receivers MUST treat envelopes with `hints.purpose: "invitation"` from unknown senders as invitations — surfacing them through the quarantine path with the sender-supplied summary, not invoking the host agent's LLM to interpret the text. Receivers MUST NOT treat the presence of `introducer_contact` as a quarantine bypass — at v0.1, vouching from the contact graph is a UI hint only, never an authentication shortcut.

Other purpose values MAY be defined in future RFCs (e.g. `"reintroduction"`, `"service-notice"`). Receivers MUST ignore unrecognized purpose values rather than failing.

## Async and offline

A long-running negotiation uses A2A `task` semantics:

- The initiator calls `message:send` (or `message:stream`); the callee returns a `taskId`.
- The initiator polls `task:get` or subscribes to status changes via `message:stream`.
- The callee's Sidecar notifies the host agent of inbound work via `social_inbox_wait` or MCP notifications (RFC-0007).

Offline peers: if SNS resolution succeeds but the endpoint is unreachable, the caller MAY:

- Retry with exponential backoff (max 30 minutes).
- Queue locally and retry on its own next tick.

The protocol does not (yet) define a relay or mailbox mechanism for fully offline delivery; that is a candidate for a future RFC.

## Errors

A Shadownet-aware error response uses HTTP + a JSON body:

```json
{
  "error":  "<machine-readable code>",
  "detail": "<human-readable message>",
  "shadownet:v": "0.1"
}
```

v0.1 error codes:

| Code | HTTP | Meaning |
| --- | --- | --- |
| `presentation_required` | 401 | No valid VP cached; supply one. |
| `presentation_invalid` | 401 | VP failed validation. |
| `level_insufficient` | 403 | Presented credentials don't satisfy the [required-level predicate](./0004-sca.md#required-level-predicates). |
| `revoked` | 403 | A presented credential is revoked. |
| `freshness_stale` | 403 | Required freshness proof missing or expired. |
| `unknown_intent` | 404 | Referenced `intentId` not known to the callee. |
| `payload_invalid` | 400 | `payload` failed validation against a known `interaction` profile the callee chose to enforce. |
| `rate_limited` | 429 | Too many requests. |
| `peer_offline` | 503 | Sidecar reachable but Subject's host agent is unreachable. |
| `peer_declined` | 403 | Recipient reviewed a quarantined invitation and chose not to accept. No further detail. |

## Schema

A normative JSON Schema for the envelope is at [`schemas/messages/envelope.schema.json`](../schemas/messages/envelope.schema.json).

## Open questions

- Whether to mandate `message:stream` (SSE) or make it a SHOULD. Some hosts struggle with long-lived HTTP.
- Whether the VP should be passed in the request body for `message:send` rather than a header, to keep large presentations off the URL/header path.
- A standard way to ask "are you online?" without committing to a full handshake (cheap presence check).
