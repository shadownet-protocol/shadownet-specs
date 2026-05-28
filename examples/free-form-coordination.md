# Free-Form Coordination — Envelope Walkthrough

A short companion to [`birthday-flow.md`](./birthday-flow.md) showing the **default** Shadownet envelope: free-form `text` with no `interaction` URI. The [Birthday flow](./birthday-flow.md) demonstrates the **typed** path (an Interaction Profile with structured payload). This document shows what changes — and what does not — when peers coordinate in plain natural language instead.

This document is **non-normative**.

## What carries over from the Birthday flow

Everything outside the envelope is identical:

- DID resolution via SNS, signed records — see Birthday flow §1–2.
- Verifiable Presentation handshake on first contact, session JWT thereafter — Birthday flow §3–5.
- A2A transport (`message:send`, `task:get`) and inbound delivery to the host agent — Birthday flow §6–8.
- SCA freshness, revocation, required-level predicates — Birthday flow §4 and RFC-0004.

The only thing this walkthrough changes is the **shape of the envelope `payload`**.

## Scenario

Sarah's Shadow pings Lukas's Shadow with an open-ended question. No calendar slots, no structured proposal — just text. Lukas's Shadow replies in kind. The two host agents (Claude Desktop and Hermes) interpret the text with their own LLMs.

## 1. MCP — Sarah's host agent calls `social_send`

```
MCP →  social_send  {
         "contactId": "ctc_lukas01",
         "payload": {
           "text": "Hey — Sarah's free Thursday or Friday next week if you want to grab dinner. No pressure on which.",
           "hints": { "deadline": "2026-05-15T17:00:00Z" }
         }
       }

MCP ←  { "intentId": "urn:uuid:int-free-001", "taskId": "task-uuid-free-001" }
```

Note the absence of `interaction`. Per [RFC-0007 §`social_send`](../rfcs/0007-mcp-tools.md), `interaction` is optional; when omitted, the Sidecar emits a free-form envelope.

## 2. A2A — Sarah's Sidecar → Lukas's Sidecar

```
POST https://lukas.example/a2a  HTTP/1.1
Authorization: Bearer <session-jwt>
X-Shadownet-Presentation: <vp-jwt>           ; first request of the session
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "message:send",
  "params": {
    "message": {
      "role": "user",
      "parts": [{
        "type": "shadownet/v1+envelope",
        "mediaType": "application/json",
        "data": {
          "shadownet:v": "0.1",
          "intentId":    "urn:uuid:int-free-001",
          "payload": {
            "text": "Hey — Sarah's free Thursday or Friday next week if you want to grab dinner. No pressure on which.",
            "hints": { "deadline": "2026-05-15T17:00:00Z" }
          }
        }
      }]
    }
  }
}
```

No `interaction` field. The envelope is still a `shadownet/v1+envelope` part — handshake, auth, correlation are unchanged.

## 3. Lukas's Sidecar — accept and surface

Per [RFC-0006 §Verifier obligations](../rfcs/0006-a2a-profile.md#verifier-obligations), Lukas's Sidecar:

1. Validates the session token and VP exactly as in the Birthday flow.
2. Parses the envelope; sees `interaction` is absent.
3. MUST accept and MUST surface the `payload` to Hermes as-is. No schema validation runs.
4. Persists the inbound message to SQLite with `intentId`, `direction:inbound`, `interaction:null`.
5. Notifies Hermes via `social_inbox_wait` (RFC-0007 §Inbound notifications).

Hermes pulls the message from `social_inbox`, feeds `payload.text` and any `hints` into its LLM along with Lukas's local context (calendar, preferences), and decides what to say.

## 4. A2A — Lukas's Sidecar → Sarah's Sidecar (response)

```
{
  "type": "shadownet/v1+envelope",
  "mediaType": "application/json",
  "data": {
    "shadownet:v": "0.1",
    "intentId":    "urn:uuid:int-free-001",
    "payload": {
      "text": "Thursday works for Lukas. He suggests somewhere quiet — last time was loud and Sarah seemed worn out by the end. Standing offer for the place near Tiergarten if she's up for it."
    }
  }
}
```

Same `intentId` correlates the reply. No `interaction`, no `hints` on the way back. Sarah's Claude Desktop reads the text and continues the conversation with Sarah in plain language.

## What's gained vs. lost

**Gained** over the typed path:

- Zero schema overhead. Two implementations with different versions of "scheduling" can still coordinate.
- Captures nuance the typed schema can't (Lukas hates BBQ smoke, Sarah is recovering from a cold). LLMs handle this natively; structured fields don't.
- No new RFC needed to add a coordination type.

**Lost** vs. the typed path:

- No machine-checkable guarantees. A free-form `text` claiming "Thursday at 8" can be misparsed across timezones; a typed `scheduling.v1` payload with an ISO-8601 instant cannot.
- Harder to write conformance tests. The typed path has a schema; the free-form path's correctness lives inside whatever LLM each side runs.
- No native capability discovery. The sender doesn't know whether the receiver "understands scheduling" — only that it can read text and reply.

## When to use which

Free-form is the default. Reach for a typed Interaction Profile only when ambiguity in the payload would actually cause harm — calendar instants, currencies, identifiers, payments, contractual terms. A coordination can also start free-form and **escalate** to typed mid-thread by including `interaction` on a later envelope in the same `intentId`.

## See also

- [RFC-0006 §Message envelope](../rfcs/0006-a2a-profile.md#message-envelope-shadownet-extensions) — normative envelope definition.
- [RFC-0007 §`social_send`](../rfcs/0007-mcp-tools.md) — MCP tool surface.
- [Birthday flow](./birthday-flow.md) — full wire-level walkthrough with the typed path.