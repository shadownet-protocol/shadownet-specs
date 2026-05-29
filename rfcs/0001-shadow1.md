_---
rfc: 0001
title: Shadownet Protocol
version: shadow1
extension: urn:shadow:v1
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet Protocol

## 1. Introduction

Shadownet is an A2A Extension that adds cryptographic per-message identity, a federated name service, and identity attestations to the A2A protocol. It is the layer that lets two personal AI agents acting on behalf of two humans address each other by a human-readable name and prove they represent the entity they claim — without inventing a new agent-to-agent transport.

Shadownet is to A2A what DKIM and DMARC are to SMTP. A Shadowname (`alice@example.com`) resolves via DNS to a provider; the provider issues a signed A2A AgentCard binding the name to a signing key; every A2A message between Shadows carries a Shadownet envelope in its extension metadata, signed by the sender's key; identity is established by credentials issued by issuers a recipient already trusts.

This document is normative. It assumes a working knowledge of A2A v1.0; references to A2A sections are to the canonical A2A specification at <https://a2a-protocol.org/>.

## 2. Conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** are interpreted per RFC 2119 / RFC 8174.

JSON is UTF-8. Where a hash or signature is computed over JSON, the canonical form is JCS per RFC 8785.

All JWS in this document use compact serialization with `alg: EdDSA` and Ed25519 keys.

Times are integer seconds since the Unix epoch unless an enclosing format (such as A2A messages, which use ISO 8601 per A2A §5.6.1) dictates otherwise. Implementations MUST tolerate ±60 s of clock skew on every time comparison.

Domain names follow RFC 1035; internationalized domains follow RFC 5891 (IDNA2008). Shadownames are case-insensitive on the local part; canonical form is lowercase.

The extension URI for this protocol is:

```
urn:shadow:v1
```

A2A allows URI identifiers for extensions; URN is a URI. The identifier does not need to resolve. Non-normative documentation MAY be hosted elsewhere and referenced by spec text. Authors MAY migrate to a `w3id.org`-hosted HTTPS URL in a future revision if specification hosting at the URI becomes desirable; the URN identifier is stable in the meantime. Future revisions bump the trailing component: `urn:shadow:v2`, etc. There are no minor versions.

## 3. Identifiers

Shadownet uses three identifier forms. All are bare strings; no DID method machinery is involved.

| Form | Example | Used for |
| --- | --- | --- |
| **Shadowname** | `alice@sh4dow.org` | The addressable agent. Always `local@provider`. |
| **Domain** | `acme.example` | Organizations and issuer endpoints. |
| **Public key** | `z6MkAlicePub...` | Multibase-encoded Ed25519 public key (base58btc, multicodec 0xed01). Self-describing via the `z6Mk` prefix. |

A Shadow is identified by its Shadowname and signs with a public key bound to that Shadowname by the provider (§5). Organizations are identified by their domain and sign with a public key published in DNS (§4.2). Credentials carry whichever form is appropriate to their kind (§6).

There is one kind of Shadow. The protocol does not type Shadows as "person" or "organization" — whatever is behind a Shadow (a verified human, an organization persona, an automated service, an AI customer-support endpoint) is conveyed by the credentials it presents, not by the shape of its identifier.

## 4. Cryptography and name service

### 4.1 Cryptography

Mandatory-to-implement, no negotiation:

| Use | Algorithm |
| --- | --- |
| Signatures | Ed25519 / EdDSA (RFC 8032). JWS compact, `alg: "EdDSA"`. |
| Hashes | SHA-256. |
| Canonical JSON | JCS (RFC 8785). |
| Transport | TLS 1.3 (RFC 8446). HTTPS everywhere; `http://localhost` permitted only for local development. |

Future revisions MAY add algorithms; v1 receivers MUST reject anything else.

### 4.2 Provider DNS record

A provider domain publishes one TXT record. There is no per-Shadowname DNS record.

```
_shadow.example.com.    IN  TXT  "v=shadow1; ep=https://shadow.example.com/v1; pk=z6MkProviderPub..."
```

Required keys:

| Key | Meaning |
| --- | --- |
| `v` | `shadow1`. |
| `ep` | Provider's HTTPS base URL. `https://` in production. |
| `pk` | Provider's signing public key (multibase Ed25519). Used to sign AgentCards (§5). |

Optional keys:

| Key | Meaning |
| --- | --- |
| `iss` | This domain operates an issuer; value is `true`. |
| `delegate` | Affiliation issuer delegation (§6.6). Multiple `delegate=` entries MAY appear. |

TXT values MAY exceed 255 characters via RFC 1035 string chaining; resolvers concatenate in order.

### 4.3 DNSSEC

DNSSEC validation is RECOMMENDED. Deployments handling `personhood` claims at high stakes SHOULD require it. v1 does not mandate DNSSEC because resolvers cannot turn it on unilaterally; without DNSSEC the risk profile equals MX-based mail.

## 5. Names

A Shadowname is resolved in two steps: one DNS lookup for the provider, and one HTTPS fetch of the signed AgentCard for the per-Shadow binding.

### 5.1 Shadowname grammar

```
shadowname  =  local "@" provider
local       =  1*63 ( ALPHA / DIGIT / "_" / "-" / "." )
provider    =  domain
```

### 5.2 AgentCard lookup

```
GET <ep>/identity/<local>
Accept: application/a2a+json
```

Response: a signed A2A AgentCard per A2A §8.4 with the Shadownet extension fields below. `Content-Type: application/a2a+json`, status `200 OK` on success.

The AgentCardSignature is by the provider key (`pk` from §4.2). The JWS header's `kid` MUST be `shadow1@<provider-domain>` (e.g., `shadow1@sh4dow.org`); verifiers MUST reject other `kid` values.

**Discovery path divergence from A2A.** Shadownet uses the per-Shadow path `<ep>/identity/<local>` rather than A2A's root-level `/.well-known/agent-card.json` (RFC 8615). The well-known URI is a single-card-per-domain construct, which cannot serve per-Shadow cards in a multi-tenant provider. Shadownet's identity-endpoint lookup is the "direct configuration" pattern from A2A §8.2 — clients are configured by the DNS+identity-endpoint discovery chain (§5.4) to find each per-Shadow card.

### 5.3 Shadownet extension fields on the AgentCard

| Field | Required | Meaning |
| --- | --- | --- |
| `shadow:v` | yes | `shadow1`. |
| `shadow:pk` | yes | The Shadow's signing public key (multibase Ed25519). |

The card's `capabilities.extensions` MUST include the Shadownet URI marked `required: true`:

```json
{
  "capabilities": {
    "extensions": [
      { "uri": "urn:shadow:v1", "required": true,
        "description": "Shadownet identity envelope" }
    ]
  }
}
```

The card's `supportedInterfaces[0].url` is the URL to which senders POST A2A `message:send` requests for this Shadow. Shadownet uses A2A's **URL-based multi-tenancy** (`multi-tenancy.md` §1) — each Shadow's URL is distinct in the AgentCard. The A2A `tenant` field is not used by this extension; receivers MUST ignore it if present.

### 5.4 Resolution flow

```
  parse alice@sh4dow.org
  DNS A query   _shadow.sh4dow.org TXT
  ◄ "v=shadow1; ep=https://shadow.sh4dow.org/v1; pk=z6MkProviderPub..."

  GET https://shadow.sh4dow.org/v1/identity/alice
  ◄ signed AgentCard (application/a2a+json)

  verify card.signatures[0] against pk
  extract shadow:pk, supportedInterfaces[0].url
```

Resolution failure (NXDOMAIN, 4xx/5xx, malformed, signature mismatch) is hard fail. Resolvers MUST surface failure distinctly from other errors so callers distinguish "unreachable" from "rejected" from "missing."

### 5.5 Caching

- DNS TXT: cached per DNS TTL.
- AgentCard: cached per the HTTP response's `Cache-Control: max-age`. RECOMMENDED `max-age=3600`. Default 3600 s if header absent.
- Providers SHOULD include an `ETag` header on AgentCard responses (typically derived from a content hash of the canonical JSON, or the card's `version` field). Resolvers SHOULD use `If-None-Match` for conditional refresh; a `304 Not Modified` extends the cached card's lifetime without redownload, per A2A §8.6.

On cache expiry, resolvers refresh. If a refresh returns a different `shadow:pk` (the Shadow rotated), envelopes signed by the old key MAY have arrived in flight; receivers SHOULD accept signatures verifiable against the previously-cached key for one additional `max-age` window after detecting rotation. This bounds split-key acceptance to two cache windows in the worst case.

### 5.6 Key rotation

A Shadow rotates by re-registering with its provider, which issues a new signed AgentCard containing the new `shadow:pk`. There is no in-band rotation statement.

Rotation latency equals the AgentCard cache TTL plus the grace window in §5.5. Credentials about the Shadowname are unaffected by key rotation (they were never bound to the key) and remain valid through rotation.

A provider rotates its own signing key by re-publishing DNS TXT with a new `pk` value. The provider MAY publish multiple `pk=` keys during a transition; verifiers MUST accept signatures from any of them.

## 6. Credentials

A credential is a JWS-compact JWT signed by an issuer, asserting one kind of identity attestation about one subject.

### 6.1 Wire shape

Header:

```json
{ "alg": "EdDSA", "typ": "shadow-cred+jwt" }
```

Payload:

```json
{
  "iss": "sca.sh4dow.org",
  "sub": "alice@sh4dow.org",
  "kind": "personhood",
  "iat": 1730000000,
  "exp": 1737776000,
  "rev": { "epoch": "2026q4", "idx": 1234 }
}
```

Required claims:

| Claim | Meaning |
| --- | --- |
| `iss` | Issuer's domain. |
| `sub` | Subject identifier — Shadowname or domain, by `kind` (§6.2). |
| `kind` | The attestation kind (§6.2). |
| `iat` | Issued-at. |
| `exp` | Expiry. |
| `rev` | Revocation pointer `{ epoch, idx }`. |

Conditional claim:

| Claim | When required | Meaning |
| --- | --- | --- |
| `org` | `kind == "org-affiliation"` | The org the subject is affiliated with (domain). |

Validation:

1. Resolve `iss` per §4.2: DNS TXT for the issuer's domain to get `pk`.
2. Verify the JWS signature.
3. Check `exp > now - 60`, `iat < now + 60`.
4. Check `typ == "shadow-cred+jwt"`.
5. Check `sub` matches the shape required by `kind`.
6. Check revocation per §6.4.
7. Check trust store (§7).

### 6.2 Kinds

Three kinds, each qualitatively different. No ordering.

| Kind | `sub` shape | Asserts |
| --- | --- | --- |
| `personhood` | Shadowname | This Shadowname represents a distinct verified human. |
| `org` | Domain | This domain is a verified organization. |
| `org-affiliation` | Shadowname (+ required `org` field) | This Shadowname acts for `org`. |

Future revisions MAY add kinds by string. Verifiers MUST treat unknown kind strings as "not present" against the trust store.

The protocol does not prescribe verification methods. An issuer's `personhood` may be email-only, ID-document-checked, biometric, or in-person; the verifier expresses what it wants by which issuers it puts in its trust store. An issuer wanting to attest at multiple quality tiers operates separate issuer domains (e.g., `id-doc.sca.sh4dow.org` vs `biometric.sca.sh4dow.org`).

### 6.3 Lifetimes

| Kind | Max `exp - iat` |
| --- | --- |
| `personhood` | 365 days |
| `org` | 365 days |
| `org-affiliation` | 30 days |

Affiliation is short-lived because affiliations change. Tighter lifetimes give tighter revocation; shorter `exp` is the only knob.

### 6.4 Revocation

Issuers publish a per-epoch status list:

```
https://<iss-domain>/.well-known/shadow/status/<epoch>
```

Body: gzip-compressed bitstring, base64url-encoded as a single ASCII string. `Content-Type: text/plain`. Honest `Cache-Control: max-age=<seconds>` (RECOMMENDED 300).

Bit `idx` set to `1` means revoked. Verifiers fetch, cache, inspect the bit at `rev.idx`.

On fetch failure or malformed list, verifiers MUST fail closed. There is no per-kind exemption: revocation is the only kill switch.

Issuers SHOULD roll the epoch when a list grows unwieldy. Old epochs MUST remain served until every credential they cover has expired.

There is no separate freshness-proof artifact. Revocation latency is bounded by `Cache-Control max-age`.

### 6.5 Issuance

The ceremony (email round-trip, ID-document, biometric, in-person, etc.) is issuer-specific and out of scope. The on-protocol boundary is CSR-in / credential-out:

```
POST https://<iss-domain>/.well-known/shadow/issue
Content-Type: application/jose

<CSR JWS>
```

CSR header:

```json
{ "alg": "EdDSA", "typ": "shadow-csr+jwt" }
```

CSR payload (signing key is the Shadow's signing key for `personhood` and `org-affiliation`; the org's signing key for `org`):

```json
{
  "iss": "alice@sh4dow.org",
  "aud": "sca.sh4dow.org",
  "iat": 1730000000,
  "exp": 1730000600,
  "req": { "kind": "personhood" }
}
```

The CSR's `iss` is the subject of the requested credential. The CSR is signed by the corresponding key: the Shadow's `shadow:pk` (from its AgentCard) for a Shadowname subject, or the org's `pk` (from its DNS) for a domain subject.

Issuer responses:

- `200 OK` (`Content-Type: application/jose`) with the credential JWS as body, when ceremony complete.
- `409 ceremony_pending` with JSON body `{ "next": "https://verify.example.com/..." }` directing the subject to complete the ceremony.
- `403 ceremony_failed` if rejected.
- `429 rate_limited` per issuer policy.

The subject re-POSTs the same CSR after ceremony completion. **Issuers SHOULD treat repeated CSRs from the same subject with identical `iss` / `aud` / `req` claims (within the lifetime of one ceremony) as idempotent** — returning the same response without re-running ceremony state. This makes client retry safe under transient network failures or split-brain polling.

Subjects MUST control their private key throughout; issuers MUST NOT request it.

### 6.6 Affiliation issuer rules

An `org-affiliation` credential's `iss` MUST be one of:

1. `iss == org`.
2. `iss` is a sub-domain of `org`'s domain (e.g., `hr.acme.example` issuing for `org=acme.example`).
3. `iss` is listed in `_shadow.<org-domain>` TXT under `delegate=` keys; multiple `delegate=` entries permitted, any match accepts.

Verifiers MUST reject `org-affiliation` credentials whose issuer satisfies none of these.

## 7. Trust

### 7.1 Trust store

A flat list of `(issuer-domain, [accepted-kinds])` tuples:

```json
[
  { "issuer": "sca.sh4dow.org", "accept": ["personhood"] },
  { "issuer": "registry.example", "accept": ["org"] },
  { "issuer": "acme.example", "accept": ["org-affiliation"] }
]
```

A credential is **trusted** iff some entry has `issuer == c.iss ∧ c.kind ∈ entry.accept`.

The reference deployment ships with one default issuer for `personhood` and one for `org`. Adding issuers is an explicit user action; there is no trust-on-first-use.

### 7.2 Acceptance policy

```json
{
  "from_contact":  [],
  "from_stranger": ["personhood", "org-affiliation"]
}
```

- `from_contact`: kinds required from a sender already in the contact graph. Empty = no credential check beyond contact membership.
- `from_stranger`: kinds required from a sender not in the contact graph. Empty = strangers rejected.

That is the entire policy surface: two lists, no compound expressions, no predicate language.

### 7.3 Evaluation

A credential set `C` satisfies the verifier with trust store `T` and required kinds `K`:

```
∃ c ∈ C :
    ∃ t ∈ T : t.issuer == c.iss ∧ c.kind ∈ t.accept
  ∧ signature(c) valid against iss key
  ∧ exp(c) > now - 60 ∧ iat(c) < now + 60
  ∧ not revoked(c)
  ∧ c.kind ∈ K
```

### 7.4 Hub contamination

Hubs (stranger-matching directories) are defended structurally, not by personhood centralization. A Hub is an organization. Hub membership is an `org-affiliation` credential. Hubs do their own vetting, issue affiliations to vetted members, and revoke for abuse.

A Hub's acceptance policy: `from_stranger: ["org-affiliation"]` with the Hub's own domain in the trust store. Bots cannot enter without first passing the Hub's vetting ceremony, whatever the Hub decides that ceremony is. The protocol does not attempt to centralize Sybil resistance; it provides the affiliation primitive and lets each closed group own its own gate.

## 8. The envelope — Shadownet A2A Extension

Every Shadownet message is an A2A request with a Shadownet envelope in extension metadata. A2A provides the transport; the envelope provides the trust.

### 8.1 A2A profile

A Shadow MUST implement the A2A HTTP+JSON binding (A2A §11), specifically `POST <agent-url>/message:send` (A2A §3.1.1), and MUST serve a signed AgentCard per §5.2 with the Shadownet extension declared `required: true`.

A Shadow MAY implement `message:stream`, `task:get`, `tasks:subscribe`, `cancelTask`, and push notifications per A2A. These are optional for v1 conformance.

Senders MUST include the A2A headers:

```
A2A-Extensions: urn:shadow:v1
A2A-Version:    1.0
Content-Type:   application/a2a+json
```

### 8.2 Envelope as message metadata

The envelope is carried as a JWS-compact string in the message's metadata under the extension URI:

```json
{
  "message": {
    "role": "ROLE_USER",
    "parts": [
      { "text": "Want to grab dinner Thursday?" }
    ],
    "messageId": "01HZ7K3CWAB4D6N5XT0M2EXAMPLE",
    "contextId": "01HZ7K2BV5R2K0DW3FCONTEXT0001",
    "extensions": ["urn:shadow:v1"],
    "metadata": {
      "urn:shadow:v1": "<envelope JWS compact string>"
    }
  }
}
```

The `extensions` array MUST list `urn:shadow:v1` to satisfy A2A §4.6.

The A2A `parts` array MUST include at least one TextPart mirroring `body.text` from the envelope; this gives non-Shadownet observers a coherent rendering. Shadownet receivers read body content from the envelope, not from parts, and use parts only for the `msg_hash` integrity check (§8.4).

Threading_: when an envelope continues a prior conversation, reuse the prior `contextId` per A2A §3.4. There is no separate `intentId` in v1; A2A's `contextId` is the correlation primitive.

### 8.3 Envelope JWS

Header:

```json
{ "alg": "EdDSA", "typ": "shadow-env+jwt", "kid": "alice@sh4dow.org" }
```

Payload:

```json
{
  "v":        "shadow1",
  "from":     "alice@sh4dow.org",
  "to":       "bob@example.org",
  "iat":      1730000050,
  "exp":      1730000350,
  "msg_hash": "sha256:<base64url>",
  "body": {
    "text":   "Want to grab dinner Thursday?",
    "intent": "urn:shadow:intent:scheduling.v1",
    "data":   { "propose": { "windows": ["2026-05-14T18:00:00Z/PT3H"] } }
  },
  "creds": ["<credential JWS>"]
}
```

Required claims:

| Claim | Meaning |
| --- | --- |
| `v` | `shadow1`. |
| `from` | Sender Shadowname (canonical). |
| `to` | Recipient Shadowname (canonical). |
| `iat` | Issued-at. |
| `exp` | Expiry. `exp - iat` MUST be ≤ 300 seconds. |
| `msg_hash` | `"sha256:" || base64url(SHA-256(canonical_message))` per §8.4. |
| `body` | Application content (§8.5). |

Conditional claim:

| Claim | When | Meaning |
| --- | --- | --- |
| `creds` | First contact, or after the receiver's cached credentials for this sender expire | Array of credential JWS strings. |

The JWS header's `kid` MUST equal `from`. Receivers MUST resolve `from` (§5.4), fetch and verify the AgentCard, and verify the envelope JWS against the AgentCard's `shadow:pk`.

### 8.4 Binding the envelope to the message

`msg_hash` binds the envelope to the A2A message it accompanies. It is computed over the canonical JSON (JCS, RFC 8785) of the message with the Shadownet extension metadata key removed:

```
canonical_input = JCS({
  "messageId":  message.messageId,
  "role":       message.role,
  "parts":      message.parts,
  "contextId":  message.contextId,           ; if present
  "taskId":     message.taskId,              ; if present
  "metadata":   message.metadata minus key "urn:shadow:v1"
})
msg_hash = "sha256:" || base64url(SHA-256(canonical_input))
```

Fields absent from the message MUST be omitted from `canonical_input` (not encoded as null).

Receivers MUST recompute `msg_hash` from the received message and compare with the envelope's value. Mismatch is hard failure.

### 8.5 Body

Three slots, all optional except `text`:

| Slot | Type | Meaning |
| --- | --- | --- |
| `text` | string | Human-readable message text. RECOMMENDED on every envelope so fallback display always makes sense. |
| `intent` | URI string | Application-defined attestation of what kind of interaction this is. Receivers MAY validate `data` against a known intent's schema; MUST NOT reject envelopes with unrecognized intents. |
| `data` | object | Application-defined structured payload, schema named by `intent` when present. |

The protocol assigns no normative intent URIs. Intent profiles (`urn:shadow:intent:scheduling.v1`, `urn:shadow:intent:intro.v1`, etc.) are defined in companion specs and evolve at their own cadence. The transport stays stable.

Receivers MUST surface envelopes with unknown `intent` values, treating `data` as opaque. Receivers MAY apply intent-specific validation only when they have opted into a known intent profile.

### 8.6 Validation

Before invoking any application logic:

1. Parse the A2A request. Confirm `A2A-Extensions` contains `urn:shadow:v1`; reject with A2A's `ExtensionSupportRequiredError` (per A2A §3.3.4) if absent.
2. Extract the envelope JWS from `message.metadata["urn:shadow:v1"]`. Reject `parse_error` if absent or non-string.
3. Parse the JWS. Confirm `typ == "shadow-env+jwt"`.
4. Validate envelope claims: `v == "shadow1"`, `to` matches the recipient served by this URL, `exp > now - 60`, `iat < now + 60`, `exp - iat ≤ 300`.
5. Resolve `from` per §5.4. Fetch and verify the sender's AgentCard. Confirm the JWS `kid` equals `from`.
6. Verify the envelope JWS signature against the AgentCard's `shadow:pk`.
7. Recompute `msg_hash` per §8.4. Compare; reject `parse_error` on mismatch.
8. Check `(from, messageId)` not in the replay cache; insert.
9. For each credential in `creds` (or in the cache for this sender if `creds` omitted): validate per §6. Cache each credential's acceptance until `exp - 60`.
10. Apply receiver policy (§9).

Any step's failure halts validation and returns the corresponding error.

### 8.7 Response

Shadownet receivers MUST return a `Message` response (not a `Task`) to `message:send`. The response Message carries `role: ROLE_AGENT` and a minimal acceptance text part. The HTTP response SHOULD include `A2A-Extensions: urn:shadow:v1` confirming extension activation (per A2A §4.6.2 extension echo).

```http
HTTP/1.1 200 OK
Content-Type: application/a2a+json
A2A-Extensions: urn:shadow:v1

{
  "message": {
    "role": "ROLE_AGENT",
    "parts": [
      { "text": "accepted" }
    ],
    "messageId": "01HZ7K4MNAB6X8K5W3T0RECEIPT01",
    "contextId": "01HZ7K2BV5R2K0DW3FCONTEXT0001"
  }
}
```

Returning a `Message` rather than a `Task` keeps Shadownet to A2A's "message-only agent" pattern (A2A `life-of-a-task.md`): the response confirms receipt; the receiver does not commit to a task state machine the sender can observe. Whether and how the receiver later replies — same `contextId`, new `messageId`, fresh envelope, opposite direction — is the receiver's choice and is conveyed as fresh inbound traffic to the sender, not as task state updates on the original message.

Receivers that need A2A `Task` semantics for application-level long-running work MAY open a fresh Task on their own initiative after acceptance. That task is its own A2A interaction, not implicitly tied to the inbound envelope.

### 8.8 Errors

A2A binding-mapped error path. In the HTTP+JSON binding, errors return `application/problem+json`:

```json
{
  "type":        "urn:shadow:error:creds-rejected",
  "title":       "Credentials rejected",
  "status":      403,
  "detail":      "No presented credential satisfies the receiver's policy.",
  "shadowError": "creds_rejected"
}
```

Defined codes:

| Code | HTTP | Meaning |
| --- | --- | --- |
| `parse_error` | 400 | A2A request, envelope JWS, payload, or `msg_hash` invalid. |
| `signature` | 401 | Envelope signature does not validate. |
| `creds_required` | 401 | No cached credentials for `from`; sender SHOULD retry with `creds`. |
| `creds_rejected` | 403 | Credentials present but none satisfy receiver policy. |
| `policy` | 403 | Receiver policy rejects this sender for undisclosed reasons. |
| `replay` | 409 | `(from, messageId)` already seen. |
| `unknown_recipient` | 404 | `to` is not served by this URL. |
| `rate_limited` | 429 | Rate limit hit. |

Error responses are deliberately coarse. Receivers MUST NOT leak whether the envelope was placed in stranger_review, whether the user has reviewed it, or any other receiver-side state.

### 8.9 Replay defense

Receivers MUST cache `(from, messageId)` for accepted envelopes for at least 10 minutes (= 2 × the maximum envelope lifetime). Replays return `replay`.

There is no per-message nonce. Replay is bounded by `exp` and the cache.

### 8.10 Offline and retry

If a recipient's `supportedInterfaces[0].url` is unreachable (connection refused, DNS NXDOMAIN, timeout), the sender's Sidecar SHOULD retry with exponential backoff: initial delay 30 s, doubling, jittered ±25%, capped at a total retry budget of 24 hours. After the budget the sender SHOULD surface the failure to its host agent and stop.

There is no relay, no MX-style secondary, no store-and-forward in v1. A recipient whose Sidecar may be unreachable operates a gateway in front: the AgentCard's `supportedInterfaces[0].url` points at an always-on gateway that accepts envelopes and forwards to the backend Sidecar by whatever internal mechanism the operator chooses. Gateway-to-backend authentication is internal to that operator and out of scope.

## 9. Receiver classification

After successful §8.6 validation, the receiver routes:

```
if from ∈ contacts and contacts[from] allows messaging:
    route = inbox
elif satisfies(creds, trust_store, policy.from_stranger):
    route = stranger_review
else:
    return error creds_rejected
```

Three routes: **inbox** (deliver normally), **stranger_review** (hold for the subject's review), **rejected** (return the error).

The protocol specifies the rule. It does not specify what the receiver does with each route. Receivers MAY auto-process stranger_review or hold it for review; receivers MAY rate-limit, batch, or drop stranger_review items after a retention window. None of these choices change the wire.

Replies do not auto-grant contact status. An envelope from a non-contact is routed as a stranger even when it shares `contextId` with a message the recipient sent.

**Same-affiliation routing (informational).** When both the sender and the recipient hold valid `org-affiliation` credentials for the same organization (the intra-org case), receivers MAY treat the sender as contact-equivalent and route to `inbox` without requiring an explicit contact-graph entry. This is the email-domain-internal pattern (`@acme.com → @acme.com` skips spam quarantine); it is RECOMMENDED as the default for enterprise deployments where members should not need to add each other to contact graphs before communicating. The wire shape is unchanged — only the receiver's classification rule.

## 10. Versioning

The protocol version appears in three places:

| Surface | Version field |
| --- | --- |
| Provider DNS TXT | `v=shadow1` |
| Envelope payload | `"v": "shadow1"` |
| AgentCard extension declaration | URI `urn:shadow:v1` |

A v2 protocol bumps to `shadow2` and to `urn:shadow:v2`. A v1 receiver MUST reject `v=shadow2` envelopes and MUST ignore v2 extension declarations. There are no per-claim version fields, no compatibility shims.

Providers transitioning between versions MAY serve both side by side; AgentCards MAY declare both extensions; clients select per their own version.

A2A's own versioning (`A2A-Version` header per A2A §3.6) is orthogonal: a Shadownet v1 message can run over A2A 1.0, 1.1, or any future minor.

## 11. Security considerations

**Agent opacity.** A2A's foundational design principle — agents are opaque, exposing capabilities but not internal state — extends to Shadownet receivers. Receivers MUST NOT signal in response variations whether an envelope was routed to inbox versus stranger_review, whether a stranger_review item has been user-reviewed, whether rate-limit budget is near exhaustion, or any other internal classification state. The coarse error vocabulary in §8.8 inherits this principle. Senders learn the disposition of their envelope only through subsequent fresh inbound from the receiver — a reply, a follow-up, or silence.

**Identity custody.** A Shadow's identity is its private key. Self-hosted Shadows hold their own key. Provider-hosted Shadows implicitly delegate key custody to the provider; the provider can sign envelopes as the Shadow and can equivocate at AgentCard issuance. This is the cloud-mail threat model. Users requiring non-custodial operation self-host. Providers MUST disclose custody posture at signup.

**DNS as the trust anchor.** The provider's authority for a domain is established by DNS. An attacker who controls DNS for `example.com` can substitute the provider's `pk` and impersonate any Shadow under that domain. DNSSEC mitigates this for validating resolvers. Without DNSSEC the risk profile equals MX-based mail.

**Provider equivocation.** A provider can serve different AgentCards to different resolvers. v1 does not solve this. A future revision MAY define a transparency log over signed AgentCards. High-stakes verifiers SHOULD compare cached cards across independent observers.

**TLS.** Every HTTPS link is TLS 1.3 in production. No STARTTLS-style negotiation. Transports that cannot do TLS 1.3 are not Shadownet transports.

**Sybil defense is structural.** A Sybil attacker must obtain N credentials at the kind the receiver requires. Each `personhood` credential costs the issuer real verification work; each `org-affiliation` credential is gated by the org's own vetting. Receivers tune `from_stranger` to the cost they want to impose, and MUST be able to rate-limit by sender Shadowname, by issuer, and by source address, and to block specific Shadownames and issuers. The protocol does not need a behavioral cost guarantee — the cost is in the credential ceremony, not in receiver compute policy.

**Replay defense.** Envelopes are bounded by `exp ≤ iat + 300` and the receiver-side `(from, messageId)` cache. Credentials are reusable for their lifetime; revocation is the kill switch.

**Trust store bootstrap.** Adding an issuer extends the attack surface to that issuer's ceremonies. Users SHOULD periodically review and prune.

**Cross-artifact confusion.** JWS `typ` headers distinguish credential (`shadow-cred+jwt`), envelope (`shadow-env+jwt`), and CSR (`shadow-csr+jwt`). Receivers MUST check `typ` matches the expected artifact.

**A2A required-extension enforcement.** Receivers' AgentCards declare the Shadownet extension `required: true`. Per A2A §3.3.4, this causes A2A to return `ExtensionSupportRequiredError` to senders that do not declare extension support; combined with `creds_rejected` on a Shadownet-aware sender presenting bad credentials, there is no path to invoke application logic without a valid envelope.

**Multi-tenant routing.** Receivers serving multiple Shadows derive the recipient from the URL path. Mismatch between URL and envelope `to` returns `unknown_recipient`, distinct from `policy` and `creds_rejected`.

**Intent surface.** Receivers that opt into validating known `body.intent` profiles MUST treat unknown intents as opaque (deliver, not reject). A receiver that rejects on unknown intents enables a sender-side reconnaissance vector for the receiver's intent registry.

## Appendix A — Channel topology

The five channels Shadownet implementations operate on:

| Pair | Protocol | Who initiates | Notes |
| --- | --- | --- | --- |
| Human ↔ Host LLM | UI-native | Either | Out of scope. |
| Host LLM ↔ Sidecar | MCP (out of this spec) | Host calls tools; Sidecar pushes events | Sidecar implementation chooses MCP server-push or host-side long-poll. Not a wire concern. |
| Sidecar ↔ DNS | DNS UDP/TCP | Sidecar | Cached per TTL. |
| Sidecar ↔ Provider HTTPS | HTTPS GET | Sidecar | AgentCard fetch. Cached per `Cache-Control`. Split-key acceptance during rotation per §5.5. |
| Sidecar ↔ Issuer HTTPS | HTTPS GET (status) / POST (CSR) | Sidecar | Status list cached per `Cache-Control`. CSR is idempotent within a ceremony per §6.5. |
| Sidecar ↔ Sidecar | A2A `message:send` | Either side, per envelope | Logical full-duplex via two POSTs: each direction is its own HTTP exchange. No persistent connection unless A2A streaming is opted into. |

Both Sidecars in any conversation MUST be reachable on an HTTPS endpoint resolvable from the public Internet (directly, behind a tunnel, or via a gateway). Async delivery is sender-side retry (§8.10); v1 has no relay.

## Appendix B — Example transaction

Alice (`alice@sh4dow.org`) sends a first-contact scheduling proposal to Bob (`bob@example.org`). Both hold `personhood` credentials from `sca.sh4dow.org`, in both trust stores.

**1. Alice's Sidecar resolves Bob.**

```
DNS:    _shadow.example.org. IN TXT  "v=shadow1; ep=https://shadow.example.org/v1; pk=z6MkBobProviderPub..."

HTTPS:  GET https://shadow.example.org/v1/identity/bob
        ◄ signed AgentCard:
          {
            "name": "Bob",
            "supportedInterfaces": [{
              "url": "https://shadow.example.org/v1/a2a/bob",
              "protocolBinding": "HTTP+JSON",
              "protocolVersion": "1.0"
            }],
            "capabilities": {
              "extensions": [
                { "uri": "urn:shadow:v1", "required": true }
              ]
            },
            "shadow:v":  "shadow1",
            "shadow:pk": "z6MkBobPub...",
            "signatures": [{
              "protected": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpPU0UiLCJraWQiOiJzaGFkb3cxQGV4YW1wbGUub3JnIn0",
              "signature": "<signature by example.org's provider pk>"
            }]
          }
```

Alice's Sidecar verifies the AgentCard signature against `z6MkBobProviderPub`. Extracts Bob's pk and A2A endpoint.

**2. Alice constructs the envelope.**

Envelope JWS header:
```json
{ "alg":"EdDSA","typ":"shadow-env+jwt","kid":"alice@sh4dow.org" }
```

Envelope payload:
```json
{
  "v":        "shadow1",
  "from":     "alice@sh4dow.org",
  "to":       "bob@example.org",
  "iat":      1730000050,
  "exp":      1730000350,
  "msg_hash": "sha256:Zk9...",
  "body": {
    "text":   "Want to grab dinner Thursday?",
    "intent": "urn:shadow:intent:scheduling.v1",
    "data":   { "propose": { "windows": ["2026-05-14T18:00:00Z/PT3H"] } }
  },
  "creds": ["<alice's personhood credential JWS>"]
}
```

`msg_hash` is over the canonical A2A message minus the Shadownet metadata key.

**3. Alice POSTs A2A `message:send` to Bob's endpoint.**

```
POST /v1/a2a/bob/message:send HTTP/1.1
Host: shadow.example.org
A2A-Version: 1.0
A2A-Extensions: urn:shadow:v1
Content-Type: application/a2a+json

{
  "message": {
    "role": "ROLE_USER",
    "parts": [
      { "text": "Want to grab dinner Thursday?" }
    ],
    "messageId": "01HZ7K3CWAB4D6N5XT0M2EXAMPLE",
    "contextId": "01HZ7K2BV5R2K0DW3FCONTEXT0001",
    "extensions": ["urn:shadow:v1"],
    "metadata": {
      "urn:shadow:v1": "<envelope JWS compact>"
    }
  }
}
```

**4. Bob's Sidecar validates.**

  1. `A2A-Extensions` includes `urn:shadow:v1`. ✓
  2. Envelope JWS present in metadata. `typ == "shadow-env+jwt"`. ✓
  3. Envelope claims valid: `to == bob@example.org`, `exp > now`, `exp - iat ≤ 300`. ✓
  4. Resolve `alice@sh4dow.org`:
     - DNS: `_shadow.sh4dow.org TXT → ep=...; pk=z6MkAliceProviderPub...`
     - HTTPS: `GET .../identity/alice → signed AgentCard`. Verify signature against `z6MkAliceProviderPub`. ✓
     - Extract `shadow:pk = z6MkAlicePub...`.
  5. JWS `kid == "alice@sh4dow.org"` matches `from`. Verify envelope signature against `z6MkAlicePub`. ✓
  6. Recompute `msg_hash` from received message minus Shadownet metadata. Matches envelope's value. ✓
  7. `(alice@sh4dow.org, 01HZ7K3...)` not in replay cache. ✓
  8. Validate `creds[0]`:
     - `iss = sca.sh4dow.org`. DNS lookup for issuer's `pk`. Verify JWS. ✓
     - `kind = "personhood"`, `sub = "alice@sh4dow.org"`. Shape check passes. ✓
     - `exp > now`. ✓
     - Fetch `https://sca.sh4dow.org/.well-known/shadow/status/2026q4` (cached). Bit `1234` is 0. ✓
     - `(sca.sh4dow.org, personhood)` in Bob's trust store. ✓
  9. Alice not in Bob's contacts. `policy.from_stranger = ["personhood", "org-affiliation"]` includes `personhood`. ✓
  10. Route: `stranger_review`. Persist envelope.

**5. Bob's Sidecar returns A2A Message response.**

```
HTTP/1.1 200 OK
Content-Type: application/a2a+json
A2A-Extensions: urn:shadow:v1

{
  "message": {
    "role": "ROLE_AGENT",
    "parts": [
      { "text": "accepted" }
    ],
    "messageId": "01HZ7K4MNAB6X8K5W3T0RECEIPT01",
    "contextId": "01HZ7K2BV5R2K0DW3FCONTEXT0001"
  }
}
```

`accepted` means the envelope was validated and routed. Whether Bob ever sees it is Bob's concern; the response leaks no receiver-side state.

**6. Bob accepts Alice into contacts.** Future envelopes from her route to inbox.

**7. Bob replies.** Bob's host LLM constructs a reply envelope (`from=bob@example.org`, `to=alice@sh4dow.org`, same `contextId`, signed by Bob's `shadow:pk`), POSTs A2A `message:send` to Alice's endpoint. Alice's Sidecar performs the symmetric validation. Bob is not in Alice's contacts; her policy routes the reply to her `stranger_review`.

## Appendix C — Wire artifact reference

The complete wire surface of v1:

1. **Provider DNS TXT.** `_shadow.<domain> IN TXT "v=shadow1; ep=...; pk=..."`. One per provider.
2. **Signed A2A AgentCard.** A2A §8 with Shadownet extension fields (`shadow:v`, `shadow:pk`). Served at `<ep>/identity/<local>`.
3. **Credential JWS.** `typ: shadow-cred+jwt`. Three kinds: `personhood`, `org`, `org-affiliation`.
4. **CSR JWS.** `typ: shadow-csr+jwt`. POSTed to `<iss-domain>/.well-known/shadow/issue`. Idempotent within a ceremony.
5. **Status list.** Gzipped bitstring at `<iss-domain>/.well-known/shadow/status/<epoch>`.
6. **Envelope JWS.** `typ: shadow-env+jwt`. Carried in A2A `message.metadata["urn:shadow:v1"]`.
7. **A2A `message:send` request** with `A2A-Extensions: urn:shadow:v1`.
8. **A2A `Message` response** (not `Task`) with `A2A-Extensions: urn:shadow:v1` echo.

## Appendix D — Out of scope (companion specifications)

This document is intentionally narrow. The following surfaces are recognized ecosystem needs that belong in separate companion documents at their own cadence and review surface:

| Companion | Scope | Notes |
| --- | --- | --- |
| Shadownet MCP Control Surface | Host-LLM ↔ Sidecar MCP tools: name resolution, contact-graph operations, envelope send, inbox / long-poll, grant management. | Planned. The wire spec defined here does not assume any particular host-LLM control surface; deployments without standardized MCP integration are conformant. |
| Shadownet Onboarding URI | `shadow://connect?ep=…&token=…` grammar for first-paste configuration of a host LLM against a Sidecar. | Planned. One URI grammar; nothing more. |
| Shadownet Intent Profiles | Application-level schemas for `body.intent` URIs (scheduling, intro, structured negotiation, etc.). | Per profile, as ecosystem demand arises. No profile blessed in v1. |

A Sidecar can claim conformance with shadow1 without implementing any of these companions. Conformance with companion specs is independent and is the subject of those specs' own conformance sections.