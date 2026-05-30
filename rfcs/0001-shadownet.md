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

Shadownet is an A2A Extension that adds cryptographic per-message identity, a federated name service, and identity attestations to the A2A protocol. It is the layer that lets two personal AI agents address each other and prove they represent the entity they claim — without inventing a new agent-to-agent transport.

A Shadow's cryptographic identity is an **Ed25519 public key**. The Shadow MAY also be addressable by a human-readable **Shadowname** (`local@provider`) that a provider binds to the key. Two addressing modes are equally valid:

- **Shadowname addressing.** `alice@sh4dow.org`. The Shadow is reachable via a name resolved through DNS and a provider-signed AgentCard. Convenient for humans; requires a provider relationship.
- **Direct addressing.** `shadow://key:z6MkAlice...@host:port`. The Shadow is reachable directly at an HTTPS endpoint, with the key embedded in the URI. No DNS, no provider. Suitable for self-hosters running a Sidecar on a VPS or home server with just a public IP.

Both modes ride the same wire (A2A `message:send` with a Shadownet envelope in extension metadata) and produce the same trust chain (key verifies the envelope signature; credentials attest to organizational affiliation). The difference is only how the receiver learns the sender's key.

This document is normative. It assumes a working knowledge of A2A v1.0; references to A2A sections are to the canonical A2A specification at <https://a2a-protocol.org/>.

## 2. Conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** are interpreted per RFC 2119 / RFC 8174.

JSON is UTF-8. Where a hash or signature is computed over JSON, the canonical form is JCS per RFC 8785.

All JWS in this document use compact serialization with `alg: EdDSA` and Ed25519 keys.

Times are integer seconds since the Unix epoch unless an enclosing format (such as A2A messages, which use ISO 8601 per A2A §5.6.1) dictates otherwise. Implementations MUST tolerate ±60 s of clock skew on every time comparison.

Domain names follow RFC 1035; internationalized domains follow RFC 5891 (IDNA2008). Shadownames are case-insensitive on the local part; canonical form is lowercase.

**Naming conventions** (applied throughout):

| Category | Convention | Examples |
| --- | --- | --- |
| JSON field names (keys) | camelCase | `msgHash`, `fromContact`, `messageId` |
| JSON value strings (kinds, error codes) | snake_case | `org_affiliation`, `creds_rejected` |
| URN suffixes and URL path segments | snake_case | `urn:shadownet:error:creds_rejected` |
| HTTP headers | Mixed-Case-With-Dashes | `A2A-Extensions`, `A2A-Version` |
| DNS TXT keys | lowercase | `v=`, `ep=`, `pk=` |
| JWT standard claims | 3-char abbreviations | `iss`, `sub`, `iat`, `exp`, `aud`, `jti`, `kid` |
| Namespaced extension fields | `shadownet:<short-key>` | `shadownet:v`, `shadownet:pk` |
| JWS `typ` header values | kebab + `+jwt` (RFC 8417) | `shadownet-env+jwt`, `shadownet-cred+jwt`, `shadownet-csr+jwt` |
| Algorithm names | standard | `EdDSA`, `SHA-256`, `TLS 1.3` |
| URI scheme | `shadow://` | `shadow://alice@sh4dow.org`, `shadow://key:z6Mk...@host:port` |

**Protocol version.** This document defines Shadownet **v0.2**. The wire and URN identifiers carry `0.2` literally:

```
urn:shadownet:0.2
```

A2A allows URI identifiers for extensions; URN is a URI. The identifier does not need to resolve. Future revisions bump the version: `urn:shadownet:0.3`, `urn:shadownet:1.0`, etc.

## 3. Identifiers

A Shadow's identity is a **public key**. Everything else — Shadownames, URIs, DNS records — exists to help other Shadows find that key.

### 3.1 Identifier forms

| Form | Example | Used for |
| --- | --- | --- |
| **Public key** | `z6MkAlicePub...` | The cryptographic identity. Multibase-encoded Ed25519 (base58btc, multicodec `0xed01`). Self-describing via the `z6Mk` prefix. |
| **Shadowname** | `alice@sh4dow.org` | Human-readable alias bound to a key by a provider. `local@provider`. |
| **Domain** | `acme.example` | Organizations, Hubs, and issuer endpoints. Identified by a DNS-resolvable hostname. |

The protocol does not type Shadows. What a Shadow represents (a human, an organization persona, an automated service) is conveyed by which credentials it presents, not by the shape of its identifier.

### 3.2 Addressing URI

Shadowname and direct addressing share the `shadow://` URI scheme. The discriminator is the userinfo slot.

```
shadow-uri      = "shadow://" userinfo "@" host [ ":" port ] [ "#" fragment ]
userinfo        = local-part / ( "key:" pubkey )
local-part      = 1*63 ( ALPHA / DIGIT / "_" / "-" / "." )
pubkey          = "z6Mk" 42*base58btc                   ; multibase Ed25519
host            = domain / ipv4 / "[" ipv6 "]"
port            = 1*DIGIT
fragment        = "sha256:" base64url-fingerprint        ; optional TLS pin
```

Parsing rule:

```
parse shadow://USERINFO@HOST[:PORT][#FRAGMENT]:
  if USERINFO contains ":":
    split on first ":" → [type, identity]
    if type == "key" → direct addressing; identity is the Ed25519 public key
    else             → reject (reserved for future addressing modes)
  else:
    USERINFO is the Shadowname local part; HOST is the provider
```

The fragment `#sha256:<base64url>` is meaningful only for direct addressing; it pins the receiver's TLS certificate fingerprint. Senders SHOULD pin on first use (TOFU) and MAY enforce a pre-shared pin when one is present.

Examples:

```
Shadowname (canonical URI):    shadow://alice@sh4dow.org
Shadowname (friendly form):    alice@sh4dow.org
Direct (URI):                  shadow://key:z6MkAlice...@192.0.2.10:8443
Direct (with TLS pin):         shadow://key:z6MkAlice...@192.0.2.10:8443#sha256:def456...
Direct (with hostname):        shadow://key:z6MkAlice...@vps.example.com:8443
```

The friendly form (`alice@sh4dow.org`, no scheme) is unambiguous because the `local@domain` shape has no other meaning in Shadownet contexts. The direct form always requires the scheme — the bare `key:z6Mk...@host:port` is not a valid Shadownet identifier on its own.

### 3.3 Wire-internal identifiers

Inside the protocol — envelope `from`/`to`, JWS `kid`, credential `sub` and `org`, trust store entries — identifiers appear in their **bare form** without the `shadow://` scheme:

| Mode | Bare form | Examples |
| --- | --- | --- |
| Shadowname | `local@provider` | `alice@sh4dow.org`, `events@acme.example` |
| Direct (key) | multibase pubkey | `z6MkAlicePub...` |

A receiver disambiguates by presence of `@`: contains `@` → Shadowname, otherwise → bare key. URI forms appear only in user-facing contexts (sharing, QR codes, configuration), never in wire artifacts.

## 4. Cryptography and name service

### 4.1 Cryptography

Mandatory-to-implement, no negotiation:

| Use | Algorithm |
| --- | --- |
| Signatures | Ed25519 / EdDSA (RFC 8032). JWS compact, `alg: "EdDSA"`. |
| Hashes | SHA-256. |
| Canonical JSON | JCS (RFC 8785). |
| Transport | TLS 1.3 (RFC 8446). HTTPS everywhere; `http://localhost` permitted only for local development. |

**TLS posture.** Two postures are valid, depending on addressing mode:

- **Shadowname-mode endpoints** use WebPKI: TLS certificates validated against the system CA store, per A2A `enterprise-ready.md`.
- **Direct-mode endpoints** use self-signed certificates with **fingerprint pinning**. Receivers present any TLS 1.3 certificate; senders pin its SHA-256 fingerprint on first use (TOFU) or against the `#sha256:...` fragment in the connection URI when supplied. The envelope JWS provides authoritative authentication regardless of TLS posture.

Direct-mode AgentCards declare the alternative posture in their `securitySchemes` field (A2A §4.5) as `"shadownet:pinned-self-signed"`, signalling to A2A-aware clients that CA-validation is intentionally not performed.

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

The provider DNS record is meaningful only for Shadowname addressing. Direct-mode Shadows do not publish DNS records.

### 4.3 DNSSEC

DNSSEC validation is RECOMMENDED for Shadowname-mode resolvers. Deployments operating affiliation-issuing domains for high-stakes Hubs SHOULD require DNSSEC on their own zone. Direct mode does not depend on DNS.

## 5. Discovery

A Shadowname is resolved through DNS and a provider-signed AgentCard. A direct-addressed Shadow is reached at its embedded endpoint and serves its own self-signed AgentCard. Both paths produce the same output: a verified public key, an A2A endpoint URL, and an AgentCard.

### 5.1 Shadowname grammar

```
shadowname  =  local "@" provider
local       =  1*63 ( ALPHA / DIGIT / "_" / "-" / "." )
provider    =  domain
```

### 5.2 Shadowname resolution

A Shadowname `local@provider` resolves in two steps:

**Step 1 — DNS lookup.** Query `_shadownet.<provider>` TXT. Parse `v`, `ep`, `pk` (§4.2). Verify `v == "0.2"`.

**Step 2 — AgentCard fetch.**

```
GET <ep>/identity/<local>
Accept: application/a2a+json
```

Response: a signed A2A AgentCard per A2A §8.4 with the Shadownet extension fields below. `Content-Type: application/a2a+json`, status `200 OK` on success.

The AgentCardSignature is by the provider key (`pk` from step 1). The JWS header's `kid` MUST be `shadownet@<provider>` (e.g., `shadownet@sh4dow.org`); verifiers MUST reject other `kid` values.

**Discovery path note.** Shadowname mode uses `<ep>/identity/<local>` rather than A2A's root-level `/.well-known/agent-card.json` because the well-known URI is a single-card-per-domain construct and Shadowname providers are multi-tenant. This is the "direct configuration" pattern from A2A §8.2.

### 5.3 Direct resolution

A direct connection URI `shadow://key:<pubkey>@<host>[:<port>][#sha256:<pin>]` resolves in one step:

**AgentCard fetch.**

```
GET https://<host>:<port>/.well-known/agent-card.json
Accept: application/a2a+json
```

This is A2A's standard well-known agent card URI (A2A §8.2.1). Direct-mode Shadows are single-tenant per endpoint, so the canonical A2A path works naturally.

The AgentCardSignature is self-signed by the Shadow itself. The JWS header's `kid` MUST equal the `<pubkey>` from the URI. Verifiers MUST verify the AgentCard signature against the pubkey and reject if the `kid` does not match.

**TLS handling.** Before issuing the GET, the sender performs the TLS handshake against `<host>:<port>`. If the URI contains a `#sha256:<pin>` fragment, the sender MUST verify the presented certificate's SHA-256 fingerprint matches the pin and reject otherwise. If no pin is present, the sender SHOULD record the fingerprint after a successful handshake and pin against it on subsequent connections (TOFU).

### 5.4 AgentCard Shadownet extension fields

Both resolution paths produce an A2A AgentCard with these Shadownet fields:

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

The card's `supportedInterfaces[0].url` is the URL to which senders POST A2A `message:send` requests for this Shadow. Shadownet uses A2A's URL-based multi-tenancy (A2A `multi-tenancy.md` §1); the A2A `tenant` field is not used by this extension and receivers MUST ignore it if present.

For direct-mode AgentCards, `securitySchemes` MUST declare `shadownet:pinned-self-signed` so A2A clients understand the non-CA TLS posture (§4.1).

### 5.5 Caching

- DNS TXT (Shadowname mode): cached per DNS TTL.
- AgentCard (both modes): cached per the HTTP response's `Cache-Control: max-age`. RECOMMENDED `max-age=3600`. Default 3600 s if header absent.
- Providers SHOULD include an `ETag` header. Resolvers SHOULD use `If-None-Match` for conditional refresh per A2A §8.6.

On cache expiry, resolvers refresh. If a refresh returns a different `shadownet:pk` (the Shadow rotated its key), receivers SHOULD accept signatures verifiable against the previously-cached key for one additional `max-age` window after detecting rotation.

### 5.6 Key rotation

**Shadowname mode.** A Shadow rotates by re-registering with its provider, which issues a new signed AgentCard containing the new `shadownet:pk`. There is no in-band rotation statement. Rotation latency equals the AgentCard cache TTL plus the §5.5 grace window. Credentials about the Shadowname are unaffected by key rotation and remain valid through rotation.

**Direct mode.** A direct-mode Shadow's key IS its identity. Rotating the key changes the identity from peers' perspective — contacts who recorded the old `shadow://key:z6Mk...@host:port` will not validate envelopes signed by the new key. Direct-mode rotation requires re-sharing the new connection URI with contacts. v0.2 provides no in-band mechanism for this; deployments that need rotation without re-share SHOULD operate in Shadowname mode.

A provider rotates its own signing key by re-publishing DNS TXT with a new `pk` value. The provider MAY publish multiple `pk=` keys during a transition; verifiers MUST accept signatures from any of them.

## 6. Credentials

A credential is a JWS-compact JWT signed by an organization (or its delegated issuer), asserting that a Shadow is affiliated with that organization.

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
| `iss` | Issuer identifier. A domain (Shadowname-mode issuer) or a public key (direct-mode issuer / Hub). Authorized for `org` per §6.6. |
| `sub` | The Shadow being attested. A Shadowname or a public key. |
| `kind` | The attestation kind. v0.2 defines exactly one: `org_affiliation`. |
| `org` | The organization the Shadow is affiliated with. A domain or a public key. |
| `iat` | Issued-at. |
| `exp` | Expiry. |
| `rev` | Revocation pointer `{ epoch, idx }`. |

Validation:

1. Resolve `iss`: for a domain, fetch `pk` from `_shadownet.<iss>` DNS TXT; for a public key, use it directly as the verification key.
2. Verify the JWS signature.
3. Check `exp > now - 60`, `iat < now + 60`.
4. Check `typ == "shadownet-cred+jwt"`.
5. Check `kind == "org_affiliation"`. Reject unknown kinds.
6. Check `iss` is authorized to issue for `org` per §6.6.
7. Check revocation per §6.4.
8. Check trust store (§7).

### 6.2 Kinds

v0.2 defines one credential kind:

| Kind | Asserts |
| --- | --- |
| `org_affiliation` | The Shadow (`sub`) is a member of, or otherwise acts for, the organization (`org`). The strength of this attestation derives from how the issuer vets membership. |

Future revisions MAY add kinds by string. Verifiers MUST treat unknown kind strings as "not present" against the trust store.

### 6.3 Lifetimes

| Kind | Max `exp - iat` |
| --- | --- |
| `org_affiliation` | 30 days |

Shorter `exp` is the only knob for tighter revocation latency.

### 6.4 Revocation

Issuers publish a per-epoch status list.

**Domain-identified issuer:**
```
https://<iss-domain>/.well-known/shadownet/status/<epoch>
```

**Key-identified issuer:** The AgentCard for the keyed issuer MUST declare a `shadownet:statusListBase` field giving the HTTPS base URL where status lists are served. Lists are then at `<base>/<epoch>`.

Body: gzip-compressed bitstring, base64url-encoded as a single ASCII string. `Content-Type: text/plain`. `Cache-Control: max-age=<seconds>` (RECOMMENDED 300).

Bit `idx` set to `1` means revoked. Verifiers fetch, cache, inspect the bit at `rev.idx`.

On fetch failure or malformed list, verifiers MUST fail closed.

Issuers SHOULD roll the epoch when a list grows unwieldy. Old epochs MUST remain served until every credential they cover has expired.

Revocation latency is bounded by `Cache-Control max-age`.

### 6.5 Issuance

The ceremony (membership application, employer onboarding, photo verification, paid subscription, OIDC token validation, etc.) is issuer-specific and out of scope. The on-protocol boundary is CSR-in / credential-out.

**Domain-identified issuer:**
```
POST https://<iss-domain>/.well-known/shadownet/issue
```

**Key-identified issuer:** The AgentCard MUST declare a `shadownet:issueEndpoint` field giving the HTTPS URL for CSR submission.

```
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

The CSR's `iss` is the Shadow requesting attestation (Shadowname or key). The `aud` is the issuer being asked. `req.org` disambiguates which organization the affiliation is being requested for.

Issuer responses:

- `200 OK` (`Content-Type: application/jose`) with the credential JWS as body.
- `409 ceremony_pending` with JSON body `{ "next": "https://verify.example.com/..." }` directing the Subject to complete the ceremony out-of-band.
- `403 ceremony_failed` if the ceremony was rejected.
- `429 rate_limited` per issuer policy.

The Subject re-POSTs the same CSR after ceremony completion. Issuers SHOULD treat repeated CSRs from the same Subject with identical `iss` / `aud` / `req` claims as idempotent within the lifetime of one ceremony.

Subjects MUST control their private key throughout; issuers MUST NOT request it.

### 6.6 Affiliation issuer rules

An `org_affiliation` credential's `iss` is authorized to issue for `org` when at least one of the following holds:

1. `iss == org`. The org issues directly. This is the **only** authorization path available to key-identified (keyed) orgs and Hubs.
2. `iss` is a sub-domain of `org`'s domain (e.g., `hr.acme.example` issuing for `org = acme.example`). Only meaningful when both `iss` and `org` are domains.
3. `iss` is listed in `_shadownet.<org-domain>` TXT under `delegate=` keys; multiple `delegate=` entries permitted, any match accepts. Only meaningful when `org` is a domain.

Verifiers MUST reject `org_affiliation` credentials whose issuer satisfies none of these.

## 7. Trust

### 7.1 Trust store

A flat list of `(issuer, [accepted-kinds])` tuples. Issuers are domains or public keys:

```json
[
  { "issuer": "acme.example",            "accept": ["org_affiliation"] },
  { "issuer": "tiergarten-club.example", "accept": ["org_affiliation"] },
  { "issuer": "z6MkPeerHub...",          "accept": ["org_affiliation"] }
]
```

A credential is **trusted** iff some entry has `issuer == c.iss ∧ c.kind ∈ entry.accept`. The verifier also checks that `c.iss` is authorized to attest for `c.org` per §6.6.

The reference Sidecar ships with an empty trust store. There is no default issuer; there is no trust-on-first-use.

### 7.2 Acceptance policy

```json
{
  "fromContact":  [],
  "fromStranger": ["org_affiliation"]
}
```

- `fromContact`: kinds required from a sender already in the contact graph. Empty (default) = no credential check beyond contact membership.
- `fromStranger`: kinds required from a sender not in the contact graph. Empty = strangers rejected outright.

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

## 8. The envelope — Shadownet A2A Extension

Every Shadownet message is an A2A request with a Shadownet envelope in extension metadata. A2A provides the transport; the envelope provides the trust.

### 8.1 A2A profile

A Shadow MUST implement the A2A HTTP+JSON binding (A2A §11), specifically `POST <agent-url>/message:send` (A2A §3.1.1), and MUST serve a signed AgentCard per §5 with the Shadownet extension declared `required: true`.

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

The A2A `parts` array MUST include at least one TextPart mirroring `body.text` from the envelope. Shadownet receivers read body content from the envelope, not from parts, and use parts only for the `msgHash` integrity check (§8.4).

Threading: when an envelope continues a prior conversation, reuse the prior `contextId` per A2A §3.4.

### 8.3 Envelope JWS

Header:

```json
{ "alg": "EdDSA", "typ": "shadownet-env+jwt", "kid": "alice@sh4dow.org" }
```

For direct-mode senders, `kid` is the sender's public key directly:

```json
{ "alg": "EdDSA", "typ": "shadownet-env+jwt", "kid": "z6MkAlicePub..." }
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
| `from` | Sender identifier (Shadowname or bare public key). |
| `to` | Recipient identifier (Shadowname or bare public key). |
| `iat` | Issued-at. |
| `exp` | Expiry. `exp - iat` MUST be ≤ 300 seconds. |
| `msgHash` | `"sha256:" || base64url(SHA-256(canonicalMessage))` per §8.4. |
| `body` | Application content (§8.5). |

Conditional claim:

| Claim | When | Meaning |
| --- | --- | --- |
| `creds` | First contact, or after the receiver's cached credentials for this sender expire | Array of credential JWS strings. |

The JWS header's `kid` MUST equal `from`. Receivers resolve `from` per §5 (Shadowname → §5.2, bare key → §5.3) and verify the envelope JWS against the resolved or embedded public key.

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
| `text` | string | Human-readable message text. RECOMMENDED on every envelope. |
| `intent` | URI string | Application-defined identifier for the kind of interaction. Receivers MAY validate `data` against a known intent's schema; MUST NOT reject envelopes with unrecognized intents. |
| `data` | object | Application-defined structured payload, schema named by `intent` when present. |

Intent profiles are defined in companion specs. The transport stays stable.

Receivers MUST surface envelopes with unknown `intent` values, treating `data` as opaque.

### 8.6 Validation

Before invoking any application logic:

1. Parse the A2A request. Confirm `A2A-Extensions` contains `urn:shadownet:0.2`; reject with A2A's `ExtensionSupportRequiredError` (per A2A §3.3.4) if absent.
2. Extract the envelope JWS from `message.metadata["urn:shadownet:0.2"]`. Reject `parse_error` if absent or non-string.
3. Parse the JWS. Confirm `typ == "shadownet-env+jwt"`.
4. Validate envelope claims: `v == "0.2"`, `to` matches the recipient served by this URL, `exp > now - 60`, `iat < now + 60`, `exp - iat ≤ 300`.
5. Resolve `from` per §5: if Shadowname, follow §5.2; if bare key, follow §5.3. Fetch and verify the sender's AgentCard. Confirm the JWS `kid` equals `from`.
6. Verify the envelope JWS signature against the sender's public key (the resolved `shadownet:pk` for Shadownames, or `from` itself for bare keys).
7. Recompute `msgHash` per §8.4. Compare; reject `parse_error` on mismatch.
8. Check `(from, messageId)` not in the replay cache; insert.
9. For each credential in `creds` (or in the cache for this sender if `creds` omitted): validate per §6. Cache each credential's acceptance until `exp - 60`.
10. Apply receiver policy (§9).

Any step's failure halts validation and returns the corresponding error.

### 8.7 Response

Shadownet receivers MUST return a `Message` response (not a `Task`) to `message:send`. The response Message carries `role: ROLE_AGENT` and a minimal acceptance text part. The HTTP response SHOULD include `A2A-Extensions: urn:shadownet:0.2` per A2A §4.6.2 extension echo.

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

The response confirms receipt and exposes no further receiver state. Subsequent activity (a reply, a follow-up) is conveyed as fresh inbound traffic, not as task updates on the original message.

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
| `policy` | 403 | Receiver policy rejects this sender. |
| `replay` | 409 | `(from, messageId)` already seen. |
| `unknown_recipient` | 404 | `to` is not served by this URL. |
| `rate_limited` | 429 | Rate limit hit. |

Receivers MUST NOT leak receiver-side classification state through error variations.

### 8.9 Replay defense

Receivers MUST cache `(from, messageId)` for accepted envelopes for at least 10 minutes (= 2 × the maximum envelope lifetime). Replays return `replay`.

There is no per-message nonce. Replay is bounded by `exp` and the cache.

### 8.10 Offline, retry, and reachability

If a recipient's `supportedInterfaces[0].url` is unreachable, the sender's Sidecar SHOULD retry with exponential backoff: initial delay 30 s, doubling, jittered ±25%, capped at a total retry budget of 24 hours. After the budget the sender SHOULD surface the failure and stop.

**Retries re-sign.** The envelope's 5-minute expiry window (§8.3) does not extend through retries. Each retry attempt MUST re-mint the envelope with fresh `iat`, `exp`, and `messageId`, signed again with the Subject's key. Senders cache the intended `body`, `to`, and `contextId` between attempts; the signing identity and message identity are re-minted on each attempt.

**Always-on endpoints.** v0.2 requires recipient endpoints to be publicly reachable during the sender's retry budget. There is no relay, no store-and-forward at the wire layer. Intermittent hosts MUST operate behind a gateway. Gateway-to-backend authentication is out of scope.

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

Three routes: **inbox** (deliver), **stranger_review** (hold for the Subject's review), **rejected** (return the error).

**Auto-add on outbound-initiated conversations.** Receivers SHOULD auto-add a sender to the contact graph when the inbound envelope's `contextId` matches a recent outbound envelope from this Subject to that sender. Receivers MUST verify the inbound's `contextId` actually corresponds to outbound from the Subject to that specific identifier, not merely to any outbound. Sidecars MAY cap the auto-add lookback window (RECOMMENDED 7 days).

This rule MUST NOT trigger on inbound envelopes whose `contextId` does not correspond to a known outbound; that path remains stranger_review.

**Same-provider-domain shortcut.** When both Shadownames share the same provider domain AND that domain operates as a single-tenant organization deployment (provider == org), receivers MAY treat the sender as contact-equivalent and route to `inbox` without requiring an explicit `org_affiliation` credential. This shortcut is **NOT valid** for multi-tenant public providers and does not apply to direct-mode identifiers (which have no provider). Operators MUST configure whether their deployment is single-tenant-org or multi-tenant-public.

## 10. Versioning

The protocol version appears in three places:

| Surface | Version field |
| --- | --- |
| Provider DNS TXT | `v=0.2` |
| Envelope payload | `"v": "0.2"` |
| AgentCard extension declaration | URI `urn:shadownet:0.2` |

A v0.3 protocol bumps to `v=0.3` and `urn:shadownet:0.3`. A v0.2 receiver MUST reject `v=0.3` envelopes and MUST ignore v0.3 extension declarations. There are no per-claim version fields, no compatibility shims.

Providers transitioning between versions MAY serve both side by side; AgentCards MAY declare both extensions; clients select per their own version.

A2A's own versioning (`A2A-Version` header per A2A §3.6) is orthogonal: a Shadownet v0.2 message can run over A2A 1.0, 1.1, or any future minor.

## 11. Security considerations

**Agent opacity.** Receivers MUST NOT signal in response variations whether an envelope was routed to inbox versus stranger_review, whether a stranger_review item has been user-reviewed, whether rate-limit budget is near exhaustion, or any other internal classification state.

**Identity custody.** Three custody arrangements are permitted. Operators MUST disclose which tier they offer:

1. **Self-hosted (Shadowname).** Subject runs their own Sidecar and operates their own provider domain. Subject controls DNS, the AgentCard signing key, and the Shadow's signing key.
2. **Hybrid (BYO-key).** Provider hosts and signs the AgentCard, binding the Shadowname to a key the Subject generated and holds. Subject runs their own Sidecar. The provider can equivocate at AgentCard issuance but cannot sign envelopes as the Subject.
3. **Fully managed.** Provider hosts the AgentCard, holds the Subject's private key, and runs the Sidecar.
4. **Self-hosted (direct).** Subject runs their own Sidecar, generates their own key, self-signs their own AgentCard, exposes it at their own endpoint. No provider relationship of any kind.

**TLS in direct mode.** The TLS layer in direct mode authenticates the *channel*, not the *identity*. The envelope JWS is the authoritative authenticator. Pin-on-first-use is acceptable because a MITM cannot forge a valid envelope JWS without the Shadow's private key. URI-supplied `#sha256:` pins are stronger because they avoid TOFU. Direct-mode Shadows SHOULD include a TLS pin in connection URIs whenever the URI is shared through a channel the sharer trusts.

**DNS as the trust anchor (Shadowname mode).** The provider's authority for a domain is established by DNS. An attacker who controls DNS for a domain can substitute the provider's `pk` and impersonate any Shadow under that domain. DNSSEC mitigates this for validating resolvers. Direct mode is not exposed to DNS attacks.

**Provider equivocation (Shadowname mode).** A provider can serve different AgentCards to different resolvers. v0.2 does not solve this. High-stakes verifiers SHOULD compare cached cards across independent observers.

**Direct-mode endpoint mobility.** A direct-mode Shadow that changes its `host:port` (e.g., VPS migration, ISP-rotated IP) breaks reachability for contacts using the old URI. Re-share the new URI through an existing channel. The key remains the identity.

**Replay defense.** Envelopes are bounded by `exp ≤ iat + 300` and the receiver-side `(from, messageId)` cache. Each retry re-mints the envelope per §8.10. Credentials are reusable for their lifetime; revocation is the kill switch.

**Trust store bootstrap.** Adding an issuer extends the attack surface to that issuer's ceremonies. Users SHOULD periodically review and prune. Default trust store ships empty.

**Cross-artifact confusion.** JWS `typ` headers distinguish credential (`shadownet-cred+jwt`), envelope (`shadownet-env+jwt`), and CSR (`shadownet-csr+jwt`). Receivers MUST check `typ` matches the expected artifact.

**A2A required-extension enforcement.** Receivers' AgentCards declare the Shadownet extension `required: true`. Per A2A §3.3.4, A2A returns `ExtensionSupportRequiredError` to senders that do not declare extension support.

**Multi-tenant routing.** Receivers serving multiple Shadowname-mode Shadows derive the recipient from the URL path. Direct-mode receivers are single-tenant per endpoint. Mismatch between URL and envelope `to` returns `unknown_recipient`.

**Auto-add abuse.** The §9 auto-add-on-outbound-initiated rule binds against `contextId` that the receiver previously generated. Receivers MUST verify the inbound's `contextId` actually corresponds to outbound from the Subject to that specific identifier.

**Intent surface.** Receivers that opt into validating known `body.intent` profiles MUST treat unknown intents as opaque. A receiver that rejects on unknown intents enables a reconnaissance vector for the receiver's intent registry.

## Appendix A — Channel topology

| Pair | Protocol | Initiator | Notes |
| --- | --- | --- | --- |
| Human ↔ Host LLM | UI-native | Either | Out of scope. |
| Host LLM ↔ Sidecar | MCP (companion spec) | Host calls tools; Sidecar pushes events | Not a wire concern. |
| Sidecar ↔ DNS | DNS UDP/TCP | Sidecar | Cached per TTL. Shadowname mode only. |
| Sidecar ↔ Provider HTTPS | HTTPS GET | Sidecar | AgentCard fetch. Shadowname mode only. |
| Sidecar ↔ Issuer HTTPS | HTTPS GET (status) / POST (CSR) | Sidecar | Status list and CSR endpoints. |
| Sidecar ↔ Sidecar | A2A `message:send` | Either side | Each direction is its own HTTP exchange. WebPKI in Shadowname mode; pinned self-signed in direct mode. |

Both Sidecars in any conversation MUST be reachable on an HTTPS endpoint resolvable from the public Internet (directly, behind a tunnel, or via a gateway). Async delivery is sender-side retry (§8.10).

## Appendix B — Example transaction

Alice (`alice@sh4dow.org`, Shadowname mode) and Bob (`shadow://key:z6MkBob...@bob-vps.example.com:8443`, direct mode) both hold `org_affiliation` credentials issued by `tiergarten-club.example`. Both have `tiergarten-club.example` in their trust store at `org_affiliation` and `fromStranger: ["org_affiliation"]`. Alice sends a first-contact message to Bob.

This example demonstrates **cross-mode** addressing: a Shadowname sender to a direct-mode recipient. The wire is identical; only the resolution path differs.

**1. Alice's Sidecar resolves Bob (direct mode).**

```
Parse:  shadow://key:z6MkBob...@bob-vps.example.com:8443
        → pubkey = z6MkBob...
        → endpoint = bob-vps.example.com:8443
        → no TLS pin (URI carries no fragment; sender will TOFU)

HTTPS:  GET https://bob-vps.example.com:8443/.well-known/agent-card.json
        TLS handshake: cert fingerprint recorded for future pinning
        ◄ signed AgentCard:
          {
            "name": "Bob",
            "supportedInterfaces": [{
              "url": "https://bob-vps.example.com:8443/a2a",
              "protocolBinding": "HTTP+JSON",
              "protocolVersion": "1.0"
            }],
            "capabilities": {
              "extensions": [
                { "uri": "urn:shadownet:0.2", "required": true }
              ]
            },
            "securitySchemes": { "shadownet:pinned-self-signed": {} },
            "shadownet:v":  "0.2",
            "shadownet:pk": "z6MkBob...",
            "signatures": [{
              "protected": "eyJhbGciOiJFZERTQSIsInR5cCI6IkpPU0UiLCJraWQiOiJ6Nk1rQm9iLi4uIn0",
              "signature": "<self-signature by z6MkBob...>"
            }]
          }
```

Alice's Sidecar verifies the AgentCard signature against the embedded `shadownet:pk` (z6MkBob...) and confirms the JWS `kid` matches the URI's pubkey.

**2. Alice constructs the envelope.**

JWS header:
```json
{ "alg":"EdDSA","typ":"shadownet-env+jwt","kid":"alice@sh4dow.org" }
```

Payload:
```json
{
  "v":       "0.2",
  "from":    "alice@sh4dow.org",
  "to":      "z6MkBob...",
  "iat":     1730000050,
  "exp":     1730000350,
  "msgHash": "sha256:Zk9...",
  "body": {
    "text":   "Hi Bob — want to grab dinner Thursday?",
    "intent": "urn:shadownet:intent:scheduling_v1",
    "data":   { "propose": { "windows": ["2026-05-14T18:00:00Z/PT3H"] } }
  },
  "creds": ["<org_affiliation credential JWS, iss=tiergarten-club.example, sub=alice@sh4dow.org>"]
}
```

Note `to` is Bob's bare key (no `shadow://` scheme, no endpoint) — those are sharing-layer artifacts; the wire carries only the identifier.

**3. Alice POSTs A2A `message:send` to Bob's endpoint.**

```
POST /a2a/message:send HTTP/1.1
Host: bob-vps.example.com:8443
A2A-Version: 1.0
A2A-Extensions: urn:shadownet:0.2
Content-Type: application/a2a+json

{
  "message": {
    "role": "ROLE_USER",
    "parts": [
      { "text": "Hi Bob — want to grab dinner Thursday?" }
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

**4. Bob's Sidecar validates** per §8.6:

  1. `A2A-Extensions` includes `urn:shadownet:0.2`. ✓
  2. Envelope JWS present in metadata. `typ == "shadownet-env+jwt"`. ✓
  3. Envelope claims valid: `to == "z6MkBob..."` (matches Bob's own key), `exp > now`, `exp - iat ≤ 300`. ✓
  4. Resolve `alice@sh4dow.org` (Shadowname mode):
     - DNS: `_shadownet.sh4dow.org TXT → ep=...; pk=z6MkAliceProviderPub...`
     - HTTPS: `GET .../identity/alice → signed AgentCard`. Signature verified. ✓
     - Extract `shadownet:pk = z6MkAlicePub...`.
  5. JWS `kid == "alice@sh4dow.org"` matches `from`. Signature verified against z6MkAlicePub. ✓
  6. `msgHash` recomputed; matches. ✓
  7. `(alice@sh4dow.org, 01HZ7K3...)` not in replay cache. ✓
  8. Credential `creds[0]` validated:
     - `iss = tiergarten-club.example`; signature verified. ✓
     - `kind = "org_affiliation"`, `sub = "alice@sh4dow.org"`, `org = "tiergarten-club.example"`. ✓
     - `iss == org` (§6.6 rule 1). ✓
     - `exp > now`; not revoked. ✓
     - `(tiergarten-club.example, org_affiliation)` in Bob's trust store. ✓
  9. Alice not in Bob's contacts. `policy.fromStranger = ["org_affiliation"]` satisfied. ✓
  10. Route: `stranger_review`.

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

**6. Bob reviews and accepts Alice into contacts.** Future envelopes from her route to inbox.

**7. Bob replies.** Bob's Sidecar constructs a reply envelope (`from="z6MkBob..."`, `to="alice@sh4dow.org"`, same `contextId`, signed by Bob's key) and POSTs A2A `message:send` to Alice's endpoint (resolved via her Shadowname). Alice's Sidecar performs symmetric validation. Bob is not in Alice's contacts, but the reply's `contextId` matches Alice's recent outbound — per §9, Alice's Sidecar adds Bob to her contact graph (recording his bare key as identifier) and routes the reply to `inbox`.

## Appendix C — Wire artifact reference

The complete wire surface of v0.2:

1. **Provider DNS TXT.** `_shadownet.<domain> IN TXT "v=0.2; ep=...; pk=..."`. One per provider. Shadowname mode only.
2. **Signed A2A AgentCard.** A2A §8 with Shadownet extension fields (`shadownet:v`, `shadownet:pk`). Served at `<ep>/identity/<local>` (Shadowname mode) or `<endpoint>/.well-known/agent-card.json` (direct mode).
3. **Credential JWS.** `typ: shadownet-cred+jwt`. One kind: `org_affiliation`. `sub`, `iss`, `org` may each be a Shadowname or a public key.
4. **CSR JWS.** `typ: shadownet-csr+jwt`. POSTed to the issuer's CSR endpoint. Idempotent within a ceremony.
5. **Status list.** Gzipped bitstring at the issuer's status list URL.
6. **Envelope JWS.** `typ: shadownet-env+jwt`. Carried in A2A `message.metadata["urn:shadownet:0.2"]`. Re-minted per retry attempt.
7. **A2A `message:send` request** with `A2A-Extensions: urn:shadownet:0.2`.
8. **A2A `Message` response** with `A2A-Extensions: urn:shadownet:0.2` echo.

## Appendix D — Companion specifications

| Companion | Scope |
| --- | --- |
| Shadownet MCP Control Surface | Host-LLM ↔ Sidecar MCP tools: name resolution, contact-graph operations, envelope send, inbox / long-poll, grant management. |
| Shadownet Onboarding URI | `shadow://connect?...` grammar for first-paste configuration of a host LLM against a Sidecar. |
| Shadownet Intent Profiles | Application-level schemas for `body.intent` URIs. |

Conformance with v0.2 does not require any companion.
