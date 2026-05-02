---
rfc: 0003
title: Shadownet Credentials
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0003: Shadownet Credentials

## Summary

Defines the credential format an SCA issues about a Shadow's [Subject](../GLOSSARY.md). Credentials are W3C [Verifiable Credentials](https://www.w3.org/TR/vc-data-model/) in JWT serialization (`vc+jwt`). Each carries a single **assurance level** claim. Verifiers consult their [trust store](../GLOSSARY.md) to decide which issuer ↔ level pairs they accept.

The issuance and trust-store mechanics are in [RFC-0004 SCA](./0004-sca.md). This RFC defines only the credential artifact.

## Levels

Levels are URI strings, not enums. v0.1 defines four:

| Level | URI | Meaning |
| --- | --- | --- |
| `L1` | `urn:shadownet:level:L1` | Email verified, rate-limited. Weak Sybil resistance. |
| `L2` | `urn:shadownet:level:L2` | Government-ID document check (e.g. Stripe Identity, Persona). |
| `L3` | `urn:shadownet:level:L3` | Unique-person check (biometric or in-person). One credential per real person, per issuer. |
| `O1` | `urn:shadownet:level:O1` | Verified organization. Subject is an org with a `did:web` DID; domain control proven at issuance. |

Future levels are added by URI; v0.1 verifiers MUST ignore unrecognised levels rather than rejecting the credential outright.

## JWT shape

A credential is a JWS-protected JWT with `typ: vc+jwt` and `alg: EdDSA`.

### Header

```json
{ "alg": "EdDSA", "typ": "vc+jwt", "kid": "did:web:sca.shadownet.example#key-1" }
```

`kid` MUST be a DID URL resolving to the issuer's signing key.

### Payload

```json
{
  "iss":   "did:web:sca.shadownet.example",
  "sub":   "did:key:z6MkSubjectPubkey...",
  "iat":   1756684800,
  "exp":   1759276800,
  "jti":   "urn:uuid:5b7c1c4a-...",
  "shadownet:v": "0.1",

  "vc": {
    "@context": [
      "https://www.w3.org/ns/credentials/v2",
      "https://shadownet.example/contexts/v1"
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
      "statusListCredential": "https://sca.shadownet.example/status/2026-q3"
    }
  }
}
```

Required claims: `iss`, `sub`, `iat`, `exp`, `jti`, `shadownet:v`, `vc.credentialSubject.{id,level,subjectType}`.

`subjectType` is `"person"` or `"organization"`. Verifiers MUST reject if `subjectType=organization` and the subject DID is not a `did:web`.

`exp - iat` SHOULD be ≤ 90 days. Long-lived credentials require a freshness proof (below) for use after the first 24 hours.

## Lifetimes and freshness

Two-tier model:

- **The credential itself** is long-lived (≤ 90 days).
- **A freshness proof** is short-lived (≤ 24 hours) and re-fetched.

A freshness proof is a small JWT issued by the same SCA, asserting that a specific credential `jti` has not been revoked as of `iat`:

```json
{
  "iss": "did:web:sca.shadownet.example",
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
