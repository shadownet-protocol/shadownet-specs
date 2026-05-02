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
    "intentId":      "urn:uuid:...",          ; correlates messages within an intent
    "sessionId":     "urn:uuid:...",          ; correlates within an A2A task
    "interaction":   "urn:shadownet:int:scheduling.v1",  ; opaque to this RFC
    "payload":       { ... }                  ; interaction-specific
  }
}
```

`interaction` is a URI naming an Interaction Profile (defined in later RFCs, e.g. scheduling, intro). v0.1 verifiers MUST NOT reject messages on the basis of an unknown `interaction`; they MAY surface them to the host agent as opaque.

`payload` is opaque to this RFC. Interaction Profiles define its schema.

## Async and offline

A long-running negotiation uses A2A `task` semantics:

- The initiator calls `message:send` (or `message:stream`); the callee returns a `taskId`.
- The initiator polls `task:get` or subscribes to status changes via `message:stream`.
- The callee's Sidecar MAY notify the host agent of inbound work via a webhook (this is `hermes-social`'s current pattern).

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
| `rate_limited` | 429 | Too many requests. |
| `peer_offline` | 503 | Sidecar reachable but Subject's host agent is unreachable. |

## Schema

A normative JSON Schema for the envelope is at [`schemas/messages/envelope.schema.json`](../schemas/messages/envelope.schema.json).

## Open questions

- Whether to mandate `message:stream` (SSE) or make it a SHOULD. Some hosts struggle with long-lived HTTP.
- Whether the VP should be passed in the request body for `message:send` rather than a header, to keep large presentations off the URL/header path.
- A standard way to ask "are you online?" without committing to a full handshake (cheap presence check).
