---
rfc: 0003
title: Shadownet Credentials
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0003: Shadownet Credentials

## Summary

Defines the credential formats an SCA (or, for affiliation, an organization) issues about a Shadow's [Subject](../GLOSSARY.md). Credentials are W3C [Verifiable Credentials](https://www.w3.org/TR/vc-data-model/) in JWT serialization (`vc+jwt`). Two credential types are defined: the **SubjectCredential** carries an assurance level, and the **AffiliationCredential** carries an organizational affiliation. Verifiers consult their [trust store](../GLOSSARY.md) to decide which issuer ↔ claim pairs they accept.

The issuance and trust-store mechanics are in [RFC-0004 SCA](./0004-sca.md). This RFC defines only the credential artifacts.

## Levels

Levels are URI strings, not enums. v0.1 defines four:

| Level | URI | Sybil-resistance method |
| --- | --- | --- |
| `L1` | `urn:shadownet:level:L1` | Email verified, rate-limited. |
| `L2` | `urn:shadownet:level:L2` | Government-ID document check (e.g. Stripe Identity, Persona). |
| `L3` | `urn:shadownet:level:L3` | Unique-person check (biometric or in-person). One credential per real person, per issuer. |
| `O1` | `urn:shadownet:level:O1` | Verified organization. Subject is an org with a `did:web` DID; domain control proven at issuance. |

Future levels are added by URI; v0.1 verifiers MUST ignore unrecognised levels rather than rejecting the credential outright.

### Levels describe Sybil resistance, not trust

A level describes **how the issuer verified that the Subject is a distinct entity** — nothing more. Verifiers MUST NOT treat levels as a trust ranking or assume any ordering among them. `L2` does NOT imply `L1`; `L3` does NOT imply `L2`; presenting a higher-numbered level confers no automatic trust beyond the Sybil-resistance method named in the level URI. Verifiers that want tier semantics ("L2 or higher") MUST enumerate accepted levels explicitly in a predicate ([RFC-0004 §Required-level predicates](./0004-sca.md#required-level-predicates)).

Trust in a peer beyond Sybil resistance is established through the contact graph and per-contact grants — see [RFC-0001 §Trust models](./0001-overview.md#trust-models) and [RFC-0007 §social_grant](./0007-mcp-tools.md#social_grant). Personhood gates strangers at the door; relational state governs everything after.

## SubjectCredential — JWT shape

The SubjectCredential is what an SCA issues to assert a level about a Subject. It is a JWS-protected JWT with `typ: vc+jwt` and `alg: EdDSA`.

### Header

```json
{ "alg": "EdDSA", "typ": "vc+jwt", "kid": "did:web:sca.sh4dow.org#key-1" }
```

`kid` MUST be a DID URL resolving to the issuer's signing key.

### Payload

```json
{
  "iss":   "did:web:sca.sh4dow.org",
  "sub":   "did:key:z6MkSubjectPubkey...",
  "iat":   1756684800,
  "exp":   1759276800,
  "jti":   "urn:uuid:5b7c1c4a-...",
  "shadownet:v": "0.1",

  "vc": {
    "@context": [
      "https://www.w3.org/ns/credentials/v2",
      "https://sh4dow.org/contexts/v1"
    ],
    "type": ["VerifiableCredential", "ShadownetSubjectCredential"],
    "credentialSubject": {
      "id":    "did:key:z6MkSubjectPubkey...",
      "level": "urn:shadownet:level:L2",
      "subjectType": "person"
    },
    "credentialStatus": {
      "type":   "BitstringStatusListEntry",
      "statusListIndex": "12345",
      "statusListCredential": "https://sca.sh4dow.org/status/2026-q3"
    }
  }
}
```

Required claims: `iss`, `sub`, `iat`, `exp`, `jti`, `shadownet:v`, `vc.credentialSubject.{id,level,subjectType}`.

`subjectType` is `"person"` or `"organization"`. Verifiers MUST reject if `subjectType=organization` and the subject DID is not a `did:web`.

`exp - iat` SHOULD be ≤ 90 days. Long-lived credentials require a freshness proof (below) for use after the first 24 hours.

## AffiliationCredential

The AffiliationCredential asserts that a Subject (typically an individual `did:key` Shadow) is affiliated with an organization (a `did:web` Shadow). It is the enterprise analog of "this person speaks for `@acme.com`."

Issuance:

- An AffiliationCredential MUST be issued either by the organization's own `did:web` DID directly, or by an SCA the organization controls (a `did:web` SCA whose domain is the org's domain or a documented sub-domain — e.g. `did:web:sca.acme.example` for `did:web:acme.example`). Verifiers MUST reject affiliation claims whose issuer cannot be authoritatively bound to the affiliation target by domain control.
- The Subject MUST be a `did:key` for individual employees/members, or a `did:web` for sub-org delegation (rare; reserved). Verifiers MUST reject if the Subject's DID shape does not match the relationship type.
- Multiple AffiliationCredentials MAY be held by the same Subject simultaneously (e.g. a contractor with affiliations at two organizations, or an employee transitioning between roles). Verifiers evaluate each affiliation independently.

### JWT shape

```json
{
  "iss":   "did:web:acme.example",
  "sub":   "did:key:z6MkEmployeePubkey...",
  "iat":   1756684800,
  "exp":   1756771200,
  "jti":   "urn:uuid:aff-7c9d...",
  "shadownet:v": "0.1",

  "vc": {
    "@context": [
      "https://www.w3.org/ns/credentials/v2",
      "https://sh4dow.org/contexts/v1"
    ],
    "type": ["VerifiableCredential", "ShadownetAffiliationCredential"],
    "credentialSubject": {
      "id":          "did:key:z6MkEmployeePubkey...",
      "affiliation": "did:web:acme.example",
      "role":        "member",
      "groups":      ["engineering", "platform-team"]
    },
    "credentialStatus": {
      "type":   "BitstringStatusListEntry",
      "statusListIndex": "0871",
      "statusListCredential": "https://acme.example/status/affiliation/2026-q3"
    }
  }
}
```

Required fields in `credentialSubject`: `id` (the Subject DID), `affiliation` (the org's `did:web` DID — MUST equal or be domain-derivable from `iss`).

Optional fields:

- `role` — operator-defined string. Common values are `member`, `admin`, `bot`, `service`. Semantics are out of scope; verifiers MAY surface the value for display.
- `groups` — operator-defined array of strings. Membership in departments, teams, or distribution lists. Used by the org's own policies (e.g. routing rules on its gateway) and surfaced to peers for context; semantics are out of scope for this RFC.

### Lifetime

`exp - iat` SHOULD be ≤ 30 days for affiliation. Affiliation changes more frequently than personhood (employees leave, contractors finish engagements, group memberships shift), and the credential's lifetime bounds the post-termination window during which a former member can still present a credential offline.

A freshness proof for an AffiliationCredential SHOULD have a `freshnessWindowSeconds` ≤ 3600 (one hour). Operators MAY tighten further. Enterprises with regulated-industry constraints (financial services, healthcare) typically require ≤ 300 s.

### Verifier acceptance

A verifier accepts an AffiliationCredential when:

1. The credential's signature validates against the issuer DID.
2. The issuer DID's domain matches (or is documented as authoritative for) `credentialSubject.affiliation`. Concretely: if `iss == credentialSubject.affiliation`, the org signed directly; if `iss != credentialSubject.affiliation`, the verifier MUST resolve the affiliation DID's document and check that the issuer DID is listed as a delegated issuer (mechanism: a `shadownet:delegatedIssuers` field in the affiliation org's DID document; see [RFC-0002 §did:web — organizations](./0002-identity.md#did-web--organizations)).
3. The verifier has chosen to grant institutional trust to the affiliation org. By default, verifiers grant institutional trust to any `did:web` org whose document resolves correctly — i.e. domain control is the trust anchor, same as `From:`-line alignment in email. Verifiers MAY maintain a deny-list or require explicit allow-listing for specific orgs.
4. Standard integrity checks pass (expiry, freshness, revocation).

When an AffiliationCredential is presented *alongside* a SubjectCredential, both are evaluated. When an AffiliationCredential is presented *without* a SubjectCredential, the verifier MAY accept the affiliation as a Sybil-resistance substitute — i.e. "Acme has verified this human" — provided the verifier's policy lists the affiliation org as an accepted personhood substitute. The reference Sidecar default is to accept any `did:web` org whose document resolves as a Sybil substitute for L1 stranger-handshake purposes; higher-stakes interactions SHOULD require an explicit allow-list entry.

## Lifetimes and freshness

Two-tier model:

- **The credential itself** is long-lived (≤ 90 days).
- **A freshness proof** is short-lived (≤ 24 hours) and re-fetched.

A freshness proof is a small JWT issued by the same SCA, asserting that a specific credential `jti` has not been revoked as of `iat`:

```json
{
  "iss": "did:web:sca.sh4dow.org",
  "sub": "urn:uuid:5b7c1c4a-...",
  "iat": 1759190400,
  "exp": 1759276800,
  "shadownet:freshness": "v1"
}
```

Verifiers:

- For credentials within 24h of `iat`: freshness proof is OPTIONAL.
- For credentials older than 24h: freshness proof is REQUIRED, and its `iat` MUST be within the window the verifier considers fresh (default 24h).

This gives revocation latency ≈ freshness window without requiring SCAs to be online for every handshake.

## Revocation

SCAs MUST publish a [BitstringStatusList](https://www.w3.org/TR/vc-bitstring-status-list/) credential at the URL named in `credentialStatus.statusListCredential`. A bit set to `1` means revoked.

Status list credentials are themselves VCs; they SHOULD be cached for 5 minutes (default; overridable by `Cache-Control`).

If a verifier cannot fetch the status list, it MUST treat the credential as invalid for any interaction requiring a level above `L1` (fail-closed for serious checks; allow optimistic continuation for low-stakes).

## Presentation

A holder presents one or more credentials inside a [Verifiable Presentation](https://www.w3.org/TR/vc-data-model/#presentations) (also `vp+jwt`):

```json
{
  "iss": "did:key:z6MkSubjectPubkey...",
  "aud": "did:key:z6MkVerifierPubkey...",
  "iat": 1759200000,
  "exp": 1759200120,
  "nonce": "<verifier-issued nonce>",
  "vp": {
    "@context": ["https://www.w3.org/ns/credentials/v2"],
    "type": ["VerifiablePresentation"],
    "verifiableCredential": [
      "<credential-jwt>",
      "<freshness-proof-jwt>"
    ]
  }
}
```

The presentation is signed by the holder (proving control of the subject DID), bound to a verifier-issued `nonce` (replay defense), and short-lived (`exp - iat` ≤ 120 s).

How presentations are exchanged in the A2A handshake is in [RFC-0006 §Handshake](./0006-a2a-profile.md#handshake).

## Schema

A normative JSON Schema for the credential payload is at [`schemas/credentials/subject-credential.schema.json`](../schemas/credentials/subject-credential.schema.json).

## Open questions

- Whether v0.1 should ship a Selective Disclosure scheme (SD-JWT). Current draft: no, defer to v0.2 — full credential disclosure is acceptable when the only claim is a level URI.
- Whether multiple levels in a single credential are useful, or always separate credentials.
