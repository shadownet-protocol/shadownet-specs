---
rfc: 0001
title: Shadownet Protocol
version: 0.2
extension: urn:shadownet:0.2
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet Protocol

## 1. Introduction

Shadownet is an A2A Extension that adds cryptographic per-message identity, a federated name service, and identity attestations to the A2A protocol. It is the layer that lets two personal AI agents acting on behalf of two humans address each other by a human-readable name and prove they represent the entity they claim — without inventing a new agent-to-agent transport.

Shadownet is to A2A what DKIM and DMARC are to SMTP. A Shadowname (`alice@example.com`) resolves via DNS to a provider; the provider issues a signed A2A AgentCard binding the name to a signing key; every A2A message between Shadows carries a Shadownet envelope in its extension metadata, signed by the sender's key; identity is established by affiliation credentials issued by organizations a recipient already trusts.

**Sybil resistance is structural, not central.** Shadownet does not define a "personhood" credential and does not designate a central authority to attest that a Shadowname represents a unique human. Cheap personhood ceremonies (email verification) don't deliver real uniqueness; strong ceremonies (biometric, in-person) bring adoption-killing liability. There is no honest middle. The protocol relocates Sybil defense to **organizations and Hubs** that can vet contextually (a dating hub checks photos; a hiring hub checks work history; a club issues membership). The single credential kind in v0.2 is `org_affiliation`.

This document is normative. It assumes a working knowledge of A2A v1.0; references to A2A sections are to the canonical A2A specification at <https://a2a-protocol.org/>.

## 2. Conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** are interpreted per RFC 2119 / RFC 8174.

JSON is UTF-8. Where a hash or signature is computed over JSON, the canonical form is JCS per RFC 8785.

All JWS in this document use compact serialization with `alg: EdDSA` and Ed25519 keys.

Times are integer seconds since the Unix epoch unless an enclosing format (such as A2A messages, which use ISO 8601 per A2A §5.6.1) dictates otherwise. Implementations MUST tolerate ±60 s of clock skew on every time comparison.

Domain names follow RFC 1035; internationalized domains follow RFC 5891 (IDNA2008). Shadownames are case-insensitive on the local part; canonical form is lowercase.

**Naming conventions.** Shadownet uses different conventions by category, applied consistently:

| Category | Convention | Examples |
| --- | --- | --- |
| JSON field names (keys) | camelCase | `msgHash`, `fromContact`, `messageId` |
| JSON value strings (kinds, error codes, grant names) | snake_case | `org_affiliation`, `creds_rejected` |
| URN suffixes and URL path segments | snake_case (matching value strings) | `urn:shadownet:error:creds_rejected` |
| HTTP headers | Mixed-Case-With-Dashes | `A2A-Extensions`, `A2A-Version` |
| DNS TXT keys | lowercase | `v=`, `ep=`, `pk=` |
| JWT standard claims | 3-char abbreviations | `iss`, `sub`, `iat`, `exp`, `aud`, `jti`, `kid` |
| Namespaced extension fields | `shadownet:<short-key>` | `shadownet:v`, `shadownet:pk` |
| JWS `typ` header values | kebab + `+jwt` (RFC 8417) | `shadownet-env+jwt`, `shadownet-cred+jwt`, `shadownet-csr+jwt` |
| Algorithm names | standard | `EdDSA`, `SHA-256`, `TLS 1.3` |

**Protocol version.** This document defines Shadownet **v0.2**, succeeding the v0.1 RFC set in this repository. v0.2 is intentionally pre-stable; breaking changes between minor versions are expected until 1.0. The wire and URN identifiers carry `0.2` literally:

```
urn:shadownet:0.2
```

A2A allows URI identifiers for extensions; URN is a URI. The identifier does not need to resolve. Non-normative documentation MAY be hosted elsewhere and referenced by spec text. Authors MAY migrate to a `w3id.org`-hosted HTTPS URL in a future revision if specification hosting at the URI becomes desirable; the URN identifier is stable in the meantime. Future revisions bump the version: `urn:shadownet:0.3`, `urn:shadownet:1.0`, etc.

## 3. Identifiers

Shadownet uses three identifier forms. All are bare strings; no DID method machinery is involved.

| Form | Example | Used for |
| --- | --- | --- |
| **Shadowname** | `alice@sh4dow.org` | The addressable agent. Always `local@provider`. |
| **Domain** | `acme.example` | Organizations, Hubs, and issuer endpoints. |
| **Public key** | `z6MkAlicePub...` | Multibase-encoded Ed25519 public key (base58btc, multicodec 0xed01). Self-describing via the `z6Mk` prefix. |

A Shadow is identified by its Shadowname and signs with a public key bound to that Shadowname by the provider (§5). Organizations and Hubs are identified by their domain and sign with a public key published in DNS (§4.2). Credentials reference Shadownames and domains directly.

There is one kind of Shadow. The protocol does not type Shadows as "person" or "organization" — whatever is behind a Shadow (a verified human, an organization persona, an automated service, an AI customer-support endpoint) is conveyed by which credentials it presents and which Hubs/orgs have admitted it, not by the shape of its identifier.

## 4. Cryptography and name service

### 4.1 Cryptography

Mandatory-to-implement, no negotiation:

| Use | Algorithm |
| --- | --- |
| Signatures | Ed25519 / EdDSA (RFC 8032). JWS compact, `alg: "EdDSA"`. |
| Hashes | SHA-256. |
| Canonical JSON | JCS (RFC 8785). |
| Transport | TLS 1.3 (RFC 8446). HTTPS everywhere; `http://localhost` permitted only for local development. |

Future revisions MAY add algorithms; v0.2 receivers MUST reject anything else.

### 4.2 Provider DNS record

A provider domain publishes one TXT record. There is no per-Shadowname DNS record.

```
_shadownet.example.com.    IN  TXT  "v=0.2; ep=https://shadow.example.com/v1; pk=z6MkProviderPub..."
```

Required keys:

| Key | Meaning |
| --- | --- |
| `v` | `0.2`. |
| `ep` | Provider's HTTPS base URL. `https://` in production. |
| `pk` | Provider's signing public key (multibase Ed25519). Used to sign AgentCards (§5). |

Optional keys:

| Key | Meaning |
| --- | --- |
| `iss` | This domain operates an issuer; value is `true`. |
| `delegate` | Affiliation issuer delegation (§6.6). Multiple `delegate=` entries MAY appear. |

TXT values MAY exceed 255 characters via RFC 1035 string chaining; resolvers concatenate in order.

### 4.3 DNSSEC

DNSSEC validation is RECOMMENDED. v0.2 does not mandate DNSSEC because resolvers cannot turn it on unilaterally; without DNSSEC the risk profile equals MX-based mail. Deployments operating affiliation-issuing domains for high-stakes Hubs SHOULD require DNSSEC on their own zone.

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

The AgentCardSignature is by the provider key (`pk` from §4.2). The JWS header's `kid` MUST be `shadownet@<provider-domain>` (e.g., `shadownet@sh4dow.org`); verifiers MUST reject other `kid` values.

**Discovery path divergence from A2A.** Shadownet uses the per-Shadow path `<ep>/identity/<local>` rather than A2A's root-level `/.well-known/agent-card.json` (RFC 8615). The well-known URI is a single-card-per-domain construct, which cannot serve per-Shadow cards in a multi-tenant provider. Shadownet's identity-endpoint lookup is the "direct configuration" pattern from A2A §8.2 — clients are configured by the DNS+identity-endpoint discovery chain (§5.4) to find each per-Shadow card.

### 5.3 Shadownet extension fields on the AgentCard

| Field | Required | Meaning |
| --- | --- | --- |
| `shadownet:v` | yes | `0.2`. |
| `shadownet:pk` | yes | The Shadow's signing public key (multibase Ed25519). |

The card's `capabilities.extensions` MUST include the Shadownet URI marked `required: true`:

```json
{
  "capabilities": {
    "extensions": [
      { "uri": "urn:shadownet:0.2", "required": true,
        "description": "Shadownet identity envelope" }
    ]
  }
}
```

The card's `supportedInterfaces[0].url` is the URL to which senders POST A2A `message:send` requests for this Shadow. Shadownet uses A2A's **URL-based multi-tenancy** (`multi-tenancy.md` §1) — each Shadow's URL is distinct in the AgentCard. The A2A `tenant` field is not used by this extension; receivers MUST ignore it if present.

### 5.4 Resolution flow

```
  parse alice@sh4dow.org
  DNS A query   _shadownet.sh4dow.org TXT
  ◄ "v=0.2; ep=https://shadow.sh4dow.org/v1; pk=z6MkProviderPub..."

  GET https://shadow.sh4dow.org/v1/identity/alice
  ◄ signed AgentCard (application/a2a+json)

  verify card.signatures[0] against pk
  extract shadownet:pk, supportedInterfaces[0].url
```

Resolution failure (NXDOMAIN, 4xx/5xx, malformed, signature mismatch) is hard fail. Resolvers MUST surface failure distinctly from other errors so callers distinguish "unreachable" from "rejected" from "missing."

### 5.5 Caching

- DNS TXT: cached per DNS TTL.
- AgentCard: cached per the HTTP response's `Cache-Control: max-age`. RECOMMENDED `max-age=3600`. Default 3600 s if header absent.
- Providers SHOULD include an `ETag` header on AgentCard responses (typically derived from a content hash of the canonical JSON, or the card's `version` field). Resolvers SHOULD use `If-None-Match` for conditional refresh; a `304 Not Modified` extends the cached card's lifetime without redownload, per A2A §8.6.

On cache expiry, resolvers refresh. If a refresh returns a different `shadownet:pk` (the Shadow rotated), envelopes signed by the old key MAY have arrived in flight; receivers SHOULD accept signatures verifiable against the previously-cached key for one additional `max-age` window after detecting rotation. This bounds split-key acceptance to two cache windows in the worst case.

### 5.6 Key rotation

A Shadow rotates by re-registering with its provider, which issues a new signed AgentCard containing the new `shadownet:pk`. There is no in-band rotation statement.

Rotation latency equals the AgentCard cache TTL plus the grace window in §5.5. Credentials about the Shadowname are unaffected by key rotation (they were never bound to the key) and remain valid through rotation.

A provider rotates its own signing key by re-publishing DNS TXT with a new `pk` value. The provider MAY publish multiple `pk=` keys during a transition; verifiers MUST accept signatures from any of them.

## 6. Credentials

A credential is a JWS-compact JWT signed by an organization (or its delegated issuer), asserting that a Shadowname is affiliated with that organization. v0.2 defines one kind of credential.

### 6.1 Wire shape

Header:

```json
{ "alg": "EdDSA", "typ": "shadownet-cred+jwt" }
```

Payload:

```json
{
  "iss": "acme.example",
  "sub": "alice@sh4dow.org",
  "kind": "org_affiliation",
  "org": "acme.example",
  "iat": 1730000000,
  "exp": 1732592000,
  "rev": { "epoch": "2026q4", "idx": 871 }
}
```

Required claims:

| Claim | Meaning |
| --- | --- |
| `iss` | Issuer's domain. Per §6.6, this is either `org` itself, a sub-domain of `org`, or a domain delegated by `org` via DNS. |
| `sub` | The Shadowname being attested. |
| `kind` | The attestation kind. v0.2 defines exactly one: `org_affiliation`. |
| `org` | The organization (domain) the Shadowname is affiliated with. |
| `iat` | Issued-at. |
| `exp` | Expiry. |
| `rev` | Revocation pointer `{ epoch, idx }`. |

Validation:

1. Resolve `iss` per §4.2: DNS TXT for the issuer's domain to get `pk`.
2. Verify the JWS signature.
3. Check `exp > now - 60`, `iat < now + 60`.
4. Check `typ == "shadownet-cred+jwt"`.
5. Check `kind == "org_affiliation"` (v0.2). Reject unknown kinds.
6. Check `iss` is authorized to issue for `org` per §6.6.
7. Check revocation per §6.4.
8. Check trust store (§7).

### 6.2 Kinds

v0.2 defines **one credential kind**:

| Kind | Asserts |
| --- | --- |
| `org_affiliation` | The Shadowname (`sub`) is a member of, or otherwise acts for, the organization (`org`). The strength of this attestation derives from how the issuer vets membership — a dating Hub verifying photos, an employer verifying employment, a club verifying paid membership. |

**No personhood, no global org-verification.** Earlier drafts and the v0.1 RFC set defined a `personhood` kind ("this Shadowname is a unique verified human") and an `org` kind ("this domain is a verified business"). Both are removed in v0.2:

- **Personhood** failed honestly. Cheap personhood ceremonies (email verification) do not deliver uniqueness — aged accounts are commodity inputs. Strong personhood ceremonies (biometric, in-person) bring Worldcoin-grade adoption resistance, liability, and operational burden. There is no honest middle. Sybil defense belongs at the Hub layer, where vetting can be contextual to the use case (§7.4). A future revision MAY re-add `personhood` if a real high-stakes need emerges; `kind` is an extensible string.
- **Global org-verification** is redundant with the trust store. If a verifier trusts `acme.example` to issue `org_affiliation`, they have already decided Acme is a real organization. An `org` credential from a third-party registrar duplicates that decision. The trust store does the work.

Future revisions MAY add kinds by string. Verifiers MUST treat unknown kind strings as "not present" against the trust store.

### 6.3 Lifetimes

| Kind | Max `exp - iat` |
| --- | --- |
| `org_affiliation` | 30 days |

Affiliation is short-lived because affiliations change (people leave employers, abandon Hubs, lose memberships). Tighter lifetimes give tighter revocation; shorter `exp` is the only knob.

### 6.4 Revocation

Issuers publish a per-epoch status list:

```
https://<iss-domain>/.well-known/shadownet/status/<epoch>
```

Body: gzip-compressed bitstring, base64url-encoded as a single ASCII string. `Content-Type: text/plain`. Honest `Cache-Control: max-age=<seconds>` (RECOMMENDED 300).

Bit `idx` set to `1` means revoked. Verifiers fetch, cache, inspect the bit at `rev.idx`.

On fetch failure or malformed list, verifiers MUST fail closed: revocation is the only kill switch.

Issuers SHOULD roll the epoch when a list grows unwieldy. Old epochs MUST remain served until every credential they cover has expired.

There is no separate freshness-proof artifact. Revocation latency is bounded by `Cache-Control max-age`.

### 6.5 Issuance

The ceremony (membership application, employer onboarding, photo verification, paid subscription, OIDC token validation, etc.) is issuer-specific and out of scope. The on-protocol boundary is CSR-in / credential-out:

```
POST https://<iss-domain>/.well-known/shadownet/issue
Content-Type: application/jose

<CSR JWS>
```

CSR header:

```json
{ "alg": "EdDSA", "typ": "shadownet-csr+jwt" }
```

CSR payload, signed by the Subject's signing key:

```json
{
  "iss": "alice@sh4dow.org",
  "aud": "acme.example",
  "iat": 1730000000,
  "exp": 1730000600,
  "req": { "kind": "org_affiliation", "org": "acme.example" }
}
```

The CSR's `iss` is the Shadowname requesting attestation. The `aud` is the issuer being asked. `req.org` disambiguates which organization the affiliation is being requested for (a delegated issuer may serve multiple orgs).

Issuer responses:

- `200 OK` (`Content-Type: application/jose`) with the credential JWS as body, when the ceremony is complete.
- `409 ceremony_pending` with JSON body `{ "next": "https://verify.example.com/..." }` directing the Subject to complete the ceremony out-of-band.
- `403 ceremony_failed` if the ceremony was rejected.
- `429 rate_limited` per issuer policy.

The Subject re-POSTs the same CSR after ceremony completion. **Issuers SHOULD treat repeated CSRs from the same Subject with identical `iss` / `aud` / `req` claims (within the lifetime of one ceremony) as idempotent** — returning the same response without re-running ceremony state. This makes client retry safe under transient network failures or split-brain polling.

Subjects MUST control their private key throughout; issuers MUST NOT request it.

**OIDC-backed issuers (non-normative).** A common pattern: an issuer wraps an OIDC flow (Sign in with Google, Sign in with Apple, Workspace SSO, GitHub OAuth) and issues `org_affiliation` based on the verified OIDC token. For Google Workspace specifically, OIDC is the strongest available proof of organizational affiliation — Google has already verified the domain controls the account. The CSR-in/cred-out shape above accommodates this without spec changes; the issuer's internal ceremony API is its own concern. OIDC is also useful as a low-friction onboarding ceremony for casual-stakes Hubs.

### 6.6 Affiliation issuer rules

An `org_affiliation` credential's `iss` MUST be one of:

1. `iss == org`. The org issues directly with its own DNS-published key.
2. `iss` is a sub-domain of `org`'s domain (e.g., `hr.acme.example` issuing for `org = acme.example`). The sub-domain is presumed under the org's control.
3. `iss` is listed in `_shadownet.<org-domain>` TXT under `delegate=` keys; multiple `delegate=` entries permitted, any match accepts. This covers outsourced identity providers (managed HR, OIDC bridges, Hub-administration services).

Verifiers MUST reject `org_affiliation` credentials whose issuer satisfies none of these.

## 7. Trust

### 7.1 Trust store

A flat list of `(issuer-domain, [accepted-kinds])` tuples:

```json
[
  { "issuer": "acme.example",            "accept": ["org_affiliation"] },
  { "issuer": "tiergarten-club.example", "accept": ["org_affiliation"] },
  { "issuer": "berlin-hiring.example",   "accept": ["org_affiliation"] }
]
```

A credential is **trusted** iff some entry has `issuer == c.iss ∧ c.kind ∈ entry.accept`. The verifier also checks that `c.iss` is authorized to attest for `c.org` per §6.6.

The reference Sidecar ships with an **empty trust store**. Users populate it as they join Hubs, take jobs, accept club memberships, or otherwise establish relationships with organizations they want incoming credentials to reference. There is no default issuer; there is no trust-on-first-use.

### 7.2 Acceptance policy

```json
{
  "fromContact":  [],
  "fromStranger": ["org_affiliation"]
}
```

- `fromContact`: kinds required from a sender already in the contact graph. Empty (default) = no credential check beyond contact membership.
- `fromStranger`: kinds required from a sender not in the contact graph. Empty = strangers rejected outright. The reference default `["org_affiliation"]` accepts strangers who present any trusted affiliation; tighten by emptying or by leaving the trust store narrow.

That is the entire policy surface: two lists, no compound expressions, no predicate language.

### 7.3 Evaluation

A credential set `C` satisfies the verifier with trust store `T` and required kinds `K`:

```
∃ c ∈ C :
    ∃ t ∈ T : t.issuer == c.iss ∧ c.kind ∈ t.accept
  ∧ c.iss authorized to issue for c.org per §6.6
  ∧ signature(c) valid against iss key
  ∧ exp(c) > now - 60 ∧ iat(c) < now + 60
  ∧ not revoked(c)
  ∧ c.kind ∈ K
```

### 7.4 Hubs and Sybil defense

Hubs are organizations that exist to introduce strangers under a contextual vetting model. A dating Hub admits members after photo verification; a hiring Hub admits after work-history checks; a regional meetup Hub admits after a paid subscription or local-presence check. Hub membership is expressed as an `org_affiliation` credential issued by the Hub.

This is the only Sybil-resistance mechanism the protocol provides. It works because Hubs vet contextually — the dating Hub knows what "real human looking for a date" looks like; a global personhood CA never could. A bot farm trying to spam a dating Hub must defeat that Hub's specific vetting per identity, which has marginal cost determined by the Hub's ceremony quality.

A Hub's own acceptance policy is the same as any other Shadow's: `fromStranger: ["org_affiliation"]` with its own domain in the trust store. Bots cannot reach Hub members without first passing the Hub's own ceremony, whatever it is. Different Hubs serve different bars.

Cross-Hub introductions, Hub-to-Hub federation, and Hub-driven coordination flows are application-level concerns built on top of the wire and out of scope for v0.2.

## 8. The envelope — Shadownet A2A Extension

Every Shadownet message is an A2A request with a Shadownet envelope in extension metadata. A2A provides the transport; the envelope provides the trust.

### 8.1 A2A profile

A Shadow MUST implement the A2A HTTP+JSON binding (A2A §11), specifically `POST <agent-url>/message:send` (A2A §3.1.1), and MUST serve a signed AgentCard per §5.2 with the Shadownet extension declared `required: true`.

A Shadow MAY implement `message:stream`, `task:get`, `tasks:subscribe`, `cancelTask`, and push notifications per A2A. These are optional for v0.2 conformance.

Senders MUST include the A2A headers:

```
A2A-Extensions: urn:shadownet:0.2
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
    "extensions": ["urn:shadownet:0.2"],
    "metadata": {
      "urn:shadownet:0.2": "<envelope JWS compact string>"
    }
  }
}
```

The `extensions` array MUST list `urn:shadownet:0.2` to satisfy A2A §4.6.

The A2A `parts` array MUST include at least one TextPart mirroring `body.text` from the envelope; this gives non-Shadownet observers a coherent rendering. Shadownet receivers read body content from the envelope, not from parts, and use parts only for the `msgHash` integrity check (§8.4).

Threading: when an envelope continues a prior conversation, reuse the prior `contextId` per A2A §3.4. There is no separate `intentId` in v0.2; A2A's `contextId` is the correlation primitive.

### 8.3 Envelope JWS

Header:

```json
{ "alg": "EdDSA", "typ": "shadownet-env+jwt", "kid": "alice@sh4dow.org" }
```

Payload:

```json
{
  "v":       "0.2",
  "from":    "alice@sh4dow.org",
  "to":      "bob@example.org",
  "iat":     1730000050,
  "exp":     1730000350,
  "msgHash": "sha256:<base64url>",
  "body": {
    "text":   "Want to grab dinner Thursday?",
    "intent": "urn:shadownet:intent:scheduling_v1",
    "data":   { "propose": { "windows": ["2026-05-14T18:00:00Z/PT3H"] } }
  },
  "creds": ["<credential JWS>"]
}
```

Required claims:

| Claim | Meaning |
| --- | --- |
| `v` | `0.2`. |
| `from` | Sender Shadowname (canonical). |
| `to` | Recipient Shadowname (canonical). |
| `iat` | Issued-at. |
| `exp` | Expiry. `exp - iat` MUST be ≤ 300 seconds. |
| `msgHash` | `"sha256:" || base64url(SHA-256(canonicalMessage))` per §8.4. |
| `body` | Application content (§8.5). |

Conditional claim:

| Claim | When | Meaning |
| --- | --- | --- |
| `creds` | First contact, or after the receiver's cached credentials for this sender expire | Array of credential JWS strings. |

The JWS header's `kid` MUST equal `from`. Receivers MUST resolve `from` (§5.4), fetch and verify the AgentCard, and verify the envelope JWS against the AgentCard's `shadownet:pk`.

### 8.4 Binding the envelope to the message

`msgHash` binds the envelope to the A2A message it accompanies. It is computed over the canonical JSON (JCS, RFC 8785) of the message with the Shadownet extension metadata key removed:

```
canonicalInput = JCS({
  "messageId":  message.messageId,
  "role":       message.role,
  "parts":      message.parts,
  "contextId":  message.contextId,           ; if present
  "taskId":     message.taskId,              ; if present
  "metadata":   message.metadata minus key "urn:shadownet:0.2"
})
msgHash = "sha256:" || base64url(SHA-256(canonicalInput))
```

Fields absent from the message MUST be omitted from `canonicalInput` (not encoded as null).

Receivers MUST recompute `msgHash` from the received message and compare with the envelope's value. Mismatch is hard failure.

### 8.5 Body

Three slots, all optional except `text`:

| Slot | Type | Meaning |
| --- | --- | --- |
| `text` | string | Human-readable message text. RECOMMENDED on every envelope so fallback display always makes sense. |
| `intent` | URI string | Application-defined attestation of what kind of interaction this is. Receivers MAY validate `data` against a known intent's schema; MUST NOT reject envelopes with unrecognized intents. |
| `data` | object | Application-defined structured payload, schema named by `intent` when present. |

The protocol assigns no normative intent URIs. Intent profiles (`urn:shadownet:intent:scheduling_v1`, `urn:shadownet:intent:intro_v1`, etc.) are defined in companion specs and evolve at their own cadence. The transport stays stable.

Receivers MUST surface envelopes with unknown `intent` values, treating `data` as opaque. Receivers MAY apply intent-specific validation only when they have opted into a known intent profile.

### 8.6 Validation

Before invoking any application logic:

1. Parse the A2A request. Confirm `A2A-Extensions` contains `urn:shadownet:0.2`; reject with A2A's `ExtensionSupportRequiredError` (per A2A §3.3.4) if absent.
2. Extract the envelope JWS from `message.metadata["urn:shadownet:0.2"]`. Reject `parse_error` if absent or non-string.
3. Parse the JWS. Confirm `typ == "shadownet-env+jwt"`.
4. Validate envelope claims: `v == "0.2"`, `to` matches the recipient served by this URL, `exp > now - 60`, `iat < now + 60`, `exp - iat ≤ 300`.
5. Resolve `from` per §5.4. Fetch and verify the sender's AgentCard. Confirm the JWS `kid` equals `from`.
6. Verify the envelope JWS signature against the AgentCard's `shadownet:pk`.
7. Recompute `msgHash` per §8.4. Compare; reject `parse_error` on mismatch.
8. Check `(from, messageId)` not in the replay cache; insert.
9. For each credential in `creds` (or in the cache for this sender if `creds` omitted): validate per §6. Cache each credential's acceptance until `exp - 60`.
10. Apply receiver policy (§9).

Any step's failure halts validation and returns the corresponding error.

### 8.7 Response

Shadownet receivers MUST return a `Message` response (not a `Task`) to `message:send`. The response Message carries `role: ROLE_AGENT` and a minimal acceptance text part. The HTTP response SHOULD include `A2A-Extensions: urn:shadownet:0.2` confirming extension activation (per A2A §4.6.2 extension echo).

```http
HTTP/1.1 200 OK
Content-Type: application/a2a+json
A2A-Extensions: urn:shadownet:0.2

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

A2A binding-mapped error path. In the HTTP+JSON binding, errors return `application/problem+json` per RFC 7807:

```json
{
  "type":   "urn:shadownet:error:creds_rejected",
  "title":  "Credentials rejected",
  "status": 403,
  "detail": "No presented credential satisfies the receiver's policy."
}
```

The `type` URN is the canonical error identifier; receivers MUST set it on every error response.

Defined error codes (each appears as `urn:shadownet:error:<code>` in the `type` field):

| Code | HTTP | Meaning |
| --- | --- | --- |
| `parse_error` | 400 | A2A request, envelope JWS, payload, or `msgHash` invalid. |
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

### 8.10 Offline, retry, and reachability

If a recipient's `supportedInterfaces[0].url` is unreachable (connection refused, DNS NXDOMAIN, timeout), the sender's Sidecar SHOULD retry with exponential backoff: initial delay 30 s, doubling, jittered ±25%, capped at a total retry budget of 24 hours. After the budget the sender SHOULD surface the failure to its host agent and stop.

**Retries re-sign.** The envelope's 5-minute expiry window (§8.3 `exp - iat ≤ 300`) does not extend through retries. Each retry attempt MUST re-mint the envelope with fresh `iat`, `exp`, and `messageId`, signed again with the Subject's key. A sender that re-sends the same envelope bytes will be rejected once `exp` passes (or earlier as a `(from, messageId)` cache entry materializes after a successful first delivery). Senders cache the intended `body`, `to`, and `contextId` between attempts; the signing identity and message identity are re-minted on each attempt.

**Always-on endpoints; intermittent hosts use the gateway pattern.** v0.2 requires recipient endpoints to be publicly reachable during the sender's retry budget. There is no relay, no MX-style secondary, no store-and-forward at the wire layer. Intermittent hosts — laptops, mobile devices, NAT-trapped home servers — MUST operate behind a gateway: the AgentCard's `supportedInterfaces[0].url` points at an always-on gateway that accepts envelopes, holds them, and delivers to the backend Sidecar via whatever internal mechanism the operator chooses (long-poll, push, SSE, queue subscription). Gateway internals — queue retention policy, backend pull protocol, gateway-to-backend authentication, abuse handling — are out of scope; reference implementations document specific approaches.

A future revision MAY define a normative provider-level store-and-forward (MX-style) to remove the always-on requirement. That work is a v0.3 candidate, not v0.2.

## 9. Receiver classification

After successful §8.6 validation, the receiver routes:

```
if from ∈ contacts and contacts[from] allows messaging:
    route = inbox
elif satisfies(creds, trustStore, policy.fromStranger):
    route = stranger_review
else:
    return error creds_rejected
```

Three routes: **inbox** (deliver normally), **stranger_review** (hold for the Subject's review), **rejected** (return the error).

The protocol specifies the rule. It does not specify what the receiver does with each route. Receivers MAY auto-process stranger_review or hold it for review; receivers MAY rate-limit, batch, or drop stranger_review items after a retention window. None of these choices change the wire.

**Auto-add on outbound-initiated conversations.** Receivers SHOULD auto-add a sender to the contact graph when the inbound envelope's `contextId` matches a recent outbound envelope from this Subject to that Shadowname. The `contextId` binding distinguishes a genuine reply from an unsolicited inbound that happens to claim a contextId — the receiver can verify that it actually issued that contextId to that recipient. Without this rule, multi-party coordination flows (one Subject initiates, several reply) require every participant to manually approve every other after their own Sidecar reached out, which kills the canonical use case. With this rule, replies to a Subject's own outbound land in `inbox` directly.

This rule SHOULD NOT trigger on inbound envelopes whose `contextId` does not correspond to a known outbound from the Subject — that path remains stranger_review.

**Same-provider-domain shortcut.** When both Shadownames share the same provider domain AND that domain operates as a single-tenant organization deployment (provider == org; e.g., both `alice@acme.example` and `bob@acme.example` where acme.example is the company's Sidecar provider), receivers MAY treat the sender as contact-equivalent and route to `inbox` without requiring an explicit `org_affiliation` credential — the shared provider IS the affiliation evidence. This shortcut is **NOT valid** for multi-tenant public providers (e.g., `sh4dow.org` hosting unrelated individuals); operators MUST configure whether their deployment is single-tenant-org or multi-tenant-public, and only enable the shortcut in the former case.

## 10. Versioning

The protocol version appears in three places:

| Surface | Version field |
| --- | --- |
| Provider DNS TXT | `v=0.2` |
| Envelope payload | `"v": "0.2"` |
| AgentCard extension declaration | URI `urn:shadownet:0.2` |

A v0.3 protocol bumps to `v=0.3` and `urn:shadownet:0.3`. A v0.2 receiver MUST reject `v=0.3` envelopes and MUST ignore v0.3 extension declarations. v0.x is pre-stable: breaking changes between minor versions are expected. There are no per-claim version fields, no compatibility shims.

Providers transitioning between versions MAY serve both side by side; AgentCards MAY declare both extensions; clients select per their own version.

A2A's own versioning (`A2A-Version` header per A2A §3.6) is orthogonal: a Shadownet v0.2 message can run over A2A 1.0, 1.1, or any future minor.

## 11. Security considerations

**Agent opacity.** A2A's foundational design principle — agents are opaque, exposing capabilities but not internal state — extends to Shadownet receivers. Receivers MUST NOT signal in response variations whether an envelope was routed to inbox versus stranger_review, whether a stranger_review item has been user-reviewed, whether rate-limit budget is near exhaustion, or any other internal classification state. The coarse error vocabulary in §8.8 inherits this principle. Senders learn the disposition of their envelope only through subsequent fresh inbound from the receiver — a reply, a follow-up, or silence.

**Identity custody — three tiers.** Shadownet permits three distinct custody arrangements with different trust models. Operators MUST disclose which tier they offer:

1. **Self-hosted.** The Subject runs their own Sidecar AND operates their own provider domain. The Subject controls DNS, the AgentCard signing key, and the Shadow's signing key. Full self-sovereignty; the only person who can sign as the Subject is the Subject. Operationally the heaviest (own domain, own TLS, own DNS hosting); trust-cheapest.

2. **Hybrid (BYO-key).** The provider hosts the AgentCard (and signs it, binding the Shadowname to a key the Subject generated and holds). The Subject runs their own Sidecar; the Subject's private key never leaves the Subject. The provider can equivocate at AgentCard issuance — serve a different `shadownet:pk` to different resolvers — but **cannot sign envelopes as the Subject**, because that requires the private key. Right for users who want a human-readable `name@hosted-provider` without surrendering signing authority. Operationally medium; trust-medium.

3. **Fully managed.** The provider hosts the AgentCard, holds the Subject's private key, and runs the Sidecar. The provider can do anything the Subject could, including signing envelopes as them. Operationally the lightest; trust-heaviest. Equivalent to the cloud-hosted mail threat model.

Providers operating tier 2 SHOULD publish a written sunset policy (export format, deletion-on-request, notice period before any wind-down) — a key the operator does not hold cannot be leaked at wind-down, and that property is the tier's main user-facing value.

**DNS as the trust anchor.** The provider's authority for a domain is established by DNS. An attacker who controls DNS for `example.com` can substitute the provider's `pk` and impersonate any Shadow under that domain. DNSSEC mitigates this for validating resolvers. Without DNSSEC the risk profile equals MX-based mail.

**Provider equivocation.** A provider can serve different AgentCards to different resolvers. v0.2 does not solve this. A future revision MAY define a transparency log over signed AgentCards. High-stakes verifiers SHOULD compare cached cards across independent observers.

**TLS.** Every HTTPS link is TLS 1.3 in production. No STARTTLS-style negotiation. Transports that cannot do TLS 1.3 are not Shadownet transports.

**Sybil defense is contextual, not central.** A Sybil attacker must obtain N credentials at the kind the receiver requires. v0.2's only kind is `org_affiliation`, and each affiliation is gated by the organization's own vetting. A dating Hub can require photo verification per identity; a hiring Hub can require work history; an employer requires actually employing someone. The cost-per-Sybil is whatever the relevant Hub's ceremony imposes, and the protocol does not attempt to pick a single global standard. Receivers narrow their trust store and `fromStranger` policy to the Hubs whose vetting they accept. The protocol does not need a behavioral cost guarantee — the cost is in the ceremony, not in receiver compute policy.

**Replay defense.** Envelopes are bounded by `exp ≤ iat + 300` and the receiver-side `(from, messageId)` cache. Each retry attempt re-mints the envelope per §8.10; senders never re-send identical bytes. Credentials are reusable for their lifetime; revocation is the kill switch.

**Trust store bootstrap.** Adding an issuer extends the attack surface to that issuer's ceremonies. Users SHOULD periodically review and prune. Default trust store ships empty; there are no protocol-mandated default issuers.

**Cross-artifact confusion.** JWS `typ` headers distinguish credential (`shadownet-cred+jwt`), envelope (`shadownet-env+jwt`), and CSR (`shadownet-csr+jwt`). Receivers MUST check `typ` matches the expected artifact.

**A2A required-extension enforcement.** Receivers' AgentCards declare the Shadownet extension `required: true`. Per A2A §3.3.4, this causes A2A to return `ExtensionSupportRequiredError` to senders that do not declare extension support; combined with `creds_rejected` on a Shadownet-aware sender presenting bad credentials, there is no path to invoke application logic without a valid envelope.

**Multi-tenant routing.** Receivers serving multiple Shadows derive the recipient from the URL path. Mismatch between URL and envelope `to` returns `unknown_recipient`, distinct from `policy` and `creds_rejected`.

**Auto-add abuse considerations.** The §9 auto-add-on-outbound-initiated rule binds against `contextId` that the receiver previously generated for outbound; an attacker cannot forge such a `contextId` without observing the original outbound. Receivers MUST verify the inbound's `contextId` actually corresponds to outbound from the Subject to that specific Shadowname, not merely to any outbound. Sidecars MAY cap the auto-add lookback window (RECOMMENDED 7 days) to limit the surface.

**Intent surface.** Receivers that opt into validating known `body.intent` profiles MUST treat unknown intents as opaque (deliver, not reject). A receiver that rejects on unknown intents enables a sender-side reconnaissance vector for the receiver's intent registry.

## Appendix A — Channel topology

The five channels Shadownet implementations operate on:

| Pair | Protocol | Who initiates | Notes |
| --- | --- | --- | --- |
| Human ↔ Host LLM | UI-native | Either | Out of scope. |
| Host LLM ↔ Sidecar | MCP (companion spec) | Host calls tools; Sidecar pushes events | Sidecar implementation chooses MCP server-push or host-side long-poll. Not a wire concern. |
| Sidecar ↔ DNS | DNS UDP/TCP | Sidecar | Cached per TTL. |
| Sidecar ↔ Provider HTTPS | HTTPS GET | Sidecar | AgentCard fetch. Cached per `Cache-Control`. Split-key acceptance during rotation per §5.5. |
| Sidecar ↔ Issuer HTTPS | HTTPS GET (status) / POST (CSR) | Sidecar | Status list cached per `Cache-Control`. CSR is idempotent within a ceremony per §6.5. |
| Sidecar ↔ Sidecar | A2A `message:send` | Either side, per envelope | Logical full-duplex via two POSTs: each direction is its own HTTP exchange. No persistent connection unless A2A streaming is opted into. |

Both Sidecars in any conversation MUST be reachable on an HTTPS endpoint resolvable from the public Internet (directly, behind a tunnel, or via a gateway). Async delivery is sender-side retry (§8.10); v0.2 has no relay.

## Appendix B — Example transaction (Hub-mediated stranger contact)

Alice (`alice@sh4dow.org`) and Bob (`bob@example.org`) are members of `tiergarten-club.example`, a Berlin tech meetup Hub. The Hub admits members after a paid subscription and an in-person verification at one of its events; on admission it issues each member an `org_affiliation` credential. Alice and Bob do not know each other and are not in each other's contact graphs.

Both have `tiergarten-club.example` in their trust store at `org_affiliation` and `fromStranger: ["org_affiliation"]`. Alice sends a first-contact scheduling proposal to Bob through the Hub-context.

**1. Alice's Sidecar resolves Bob.**

```
DNS:    _shadownet.example.org. IN TXT  "v=0.2; ep=https://shadow.example.org/v1; pk=z6MkBobProviderPub..."

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
                { "uri": "urn:shadownet:0.2", "required": true }
              ]
            },
            "shadownet:v":  "0.2",
            "shadownet:pk": "z6MkBobPub...",
            "signatures": [{
              "protected": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpPU0UiLCJraWQiOiJzaGFkb3duZXRAZXhhbXBsZS5vcmcifQ",
              "signature": "<signature by example.org's provider pk>"
            }]
          }
```

Alice's Sidecar verifies the AgentCard signature against `z6MkBobProviderPub`. Extracts Bob's pk and A2A endpoint.

**2. Alice constructs the envelope.**

Envelope JWS header:
```json
{ "alg":"EdDSA","typ":"shadownet-env+jwt","kid":"alice@sh4dow.org" }
```

Envelope payload (with Alice's Hub affiliation credential):
```json
{
  "v":       "0.2",
  "from":    "alice@sh4dow.org",
  "to":      "bob@example.org",
  "iat":     1730000050,
  "exp":     1730000350,
  "msgHash": "sha256:Zk9...",
  "body": {
    "text":   "Hi Bob — saw you at Tuesday's meetup. Want to grab dinner Thursday?",
    "intent": "urn:shadownet:intent:scheduling_v1",
    "data":   { "propose": { "windows": ["2026-05-14T18:00:00Z/PT3H"] } }
  },
  "creds": ["<alice's org_affiliation credential JWS, iss=tiergarten-club.example, org=tiergarten-club.example>"]
}
```

`msgHash` is over the canonical A2A message minus the Shadownet metadata key.

**3. Alice POSTs A2A `message:send` to Bob's endpoint.**

```
POST /v1/a2a/bob/message:send HTTP/1.1
Host: shadow.example.org
A2A-Version: 1.0
A2A-Extensions: urn:shadownet:0.2
Content-Type: application/a2a+json

{
  "message": {
    "role": "ROLE_USER",
    "parts": [
      { "text": "Hi Bob — saw you at Tuesday's meetup. Want to grab dinner Thursday?" }
    ],
    "messageId": "01HZ7K3CWAB4D6N5XT0M2EXAMPLE",
    "contextId": "01HZ7K2BV5R2K0DW3FCONTEXT0001",
    "extensions": ["urn:shadownet:0.2"],
    "metadata": {
      "urn:shadownet:0.2": "<envelope JWS compact>"
    }
  }
}
```

**4. Bob's Sidecar validates.**

  1. `A2A-Extensions` includes `urn:shadownet:0.2`. ✓
  2. Envelope JWS present in metadata. `typ == "shadownet-env+jwt"`. ✓
  3. Envelope claims valid: `to == bob@example.org`, `exp > now`, `exp - iat ≤ 300`. ✓
  4. Resolve `alice@sh4dow.org`:
     - DNS: `_shadownet.sh4dow.org TXT → ep=...; pk=z6MkAliceProviderPub...`
     - HTTPS: `GET .../identity/alice → signed AgentCard`. Verify signature against `z6MkAliceProviderPub`. ✓
     - Extract `shadownet:pk = z6MkAlicePub...`.
  5. JWS `kid == "alice@sh4dow.org"` matches `from`. Verify envelope signature against `z6MkAlicePub`. ✓
  6. Recompute `msgHash` from received message minus Shadownet metadata. Matches envelope's value. ✓
  7. `(alice@sh4dow.org, 01HZ7K3...)` not in replay cache. ✓
  8. Validate `creds[0]`:
     - `iss = tiergarten-club.example`. DNS lookup for issuer's `pk`. Verify JWS. ✓
     - `kind = "org_affiliation"`, `sub = "alice@sh4dow.org"`, `org = "tiergarten-club.example"`. ✓
     - `iss == org` (§6.6 rule 1). ✓
     - `exp > now`. ✓
     - Fetch `https://tiergarten-club.example/.well-known/shadownet/status/2026q4` (cached). Bit `0871` is 0. ✓
     - `(tiergarten-club.example, org_affiliation)` in Bob's trust store. ✓
  9. Alice not in Bob's contacts. `policy.fromStranger = ["org_affiliation"]`; the credential satisfies. ✓
  10. Route: `stranger_review` (Alice is a stranger who passed the Hub's vetting; Bob's Subject reviews). Persist envelope.

**5. Bob's Sidecar returns A2A Message response.**

```
HTTP/1.1 200 OK
Content-Type: application/a2a+json
A2A-Extensions: urn:shadownet:0.2

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

**6. Bob's host LLM reviews and Bob accepts Alice into contacts.** Future envelopes from her route to inbox.

**7. Bob replies.** Bob's host LLM constructs a reply envelope (`from=bob@example.org`, `to=alice@sh4dow.org`, same `contextId`, signed by Bob's `shadownet:pk`), POSTs A2A `message:send` to Alice's endpoint. Alice's Sidecar performs the symmetric validation. Bob is not in Alice's contacts — but the reply's `contextId` matches Alice's own recent outbound. Per §9's auto-add rule, Alice's Sidecar adds Bob to her contact graph and routes the reply to `inbox` directly. The cross-Sidecar approval round-trip is skipped because the contextId binding proves this is a genuine reply.

## Appendix C — Wire artifact reference

The complete wire surface of v0.2:

1. **Provider DNS TXT.** `_shadownet.<domain> IN TXT "v=0.2; ep=...; pk=..."`. One per provider.
2. **Signed A2A AgentCard.** A2A §8 with Shadownet extension fields (`shadownet:v`, `shadownet:pk`). Served at `<ep>/identity/<local>`.
3. **Credential JWS.** `typ: shadownet-cred+jwt`. One kind in v0.2: `org_affiliation`.
4. **CSR JWS.** `typ: shadownet-csr+jwt`. POSTed to `<iss-domain>/.well-known/shadownet/issue`. Idempotent within a ceremony.
5. **Status list.** Gzipped bitstring at `<iss-domain>/.well-known/shadownet/status/<epoch>`.
6. **Envelope JWS.** `typ: shadownet-env+jwt`. Carried in A2A `message.metadata["urn:shadownet:0.2"]`. Re-minted per retry attempt.
7. **A2A `message:send` request** with `A2A-Extensions: urn:shadownet:0.2`.
8. **A2A `Message` response** (not `Task`) with `A2A-Extensions: urn:shadownet:0.2` echo.

## Appendix D — Companion specifications

This document is intentionally narrow. The following surfaces live in separate companion documents at their own cadence:

| Companion | Scope |
| --- | --- |
| Shadownet MCP Control Surface | Host-LLM ↔ Sidecar MCP tools: name resolution, contact-graph operations, envelope send, inbox / long-poll, grant management. |
| Shadownet Onboarding URI | `shadow://connect?ep=…&token=…` grammar for first-paste configuration of a host LLM against a Sidecar. |
| Shadownet Intent Profiles | Application-level schemas for `body.intent` URIs (scheduling, intro, structured negotiation, etc.). |

Candidate v0.3 work, not v0.2:

- **Keyed (domainless) addressing mode.** A second addressing path where identity is the Ed25519 public key, the endpoint is `key@ip:port` with a pinned TLS fingerprint, and no DNS or CA is required. Authenticity from the envelope JWS, confidentiality from TLS key-pinning. Lets self-hosters skip "buy a domain, set DNS, get a cert."
- **Provider-level store-and-forward.** A minimal MX-style relay at the provider so intermittent hosts (laptops, mobile devices) work without operating a gateway. Requires honest design of queue retention, abuse prevention, and gateway-to-backend pull semantics.

A Sidecar can claim conformance with v0.2 without implementing any of these companions or v0.3 candidates. Conformance with companion specs is independent and is the subject of those specs' own conformance sections.