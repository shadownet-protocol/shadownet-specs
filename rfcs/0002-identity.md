---
rfc: 0002
title: Shadow Identity
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0002: Shadow Identity

## Summary

A Shadow's identity is a [DID](../GLOSSARY.md). v0.1 specifies two DID methods: `did:key` for individuals and `did:web` for organizations. All other Shadownet artifacts (credentials, A2A handshake, SNS records) reference DIDs.

## DID methods

### `did:key` — individuals

A Shadow representing a human MUST use a [`did:key`](https://w3c-ccg.github.io/did-method-key/) DID derived from an Ed25519 public key.

```
did:key:z6Mk<multibase-encoded-Ed25519-public-key>
```

Properties:

- **Self-certifying.** The DID *is* the public key (multibase-encoded with multicodec `0xed01`). No registry, no resolution call.
- **Unforgeable.** Holding the matching private key is the only way to act as the DID.
- **Ephemeral by intent.** A new DID can be generated locally at any time.

### `did:web` — organizations

A Shadow representing an organization MUST use a [`did:web`](https://w3c-ccg.github.io/did-method-web/) DID anchored at a domain the organization controls.

```
did:web:acme.example
```

The DID document MUST be served at `https://acme.example/.well-known/did.json` over TLS 1.3, MUST list at least one Ed25519 verification method, and MUST be ≤ 16 KiB.

Domain control proves organizational identity at the same level the SSL ecosystem already trusts it.

### Forbidden

A Shadow representing a human MUST NOT use `did:web`. A Shadow representing an organization MUST NOT use `did:key` (orgs need transferable, domain-anchored identity).

## Key generation

Private keys MUST be generated with a CSPRNG. Implementations SHOULD use the platform key-storage facility (Keychain on macOS, Secret Service on Linux, DPAPI on Windows) when available.

The current shadownet-local Sidecar persists `private.key` / `public.key` files at mode `0o600` under the data directory. v0.1 Sidecars MAY do the same; SHOULD migrate to platform keystore in subsequent versions.

## Key rotation

DIDs in v0.1 are not rotatable in place. A Shadow that needs to rotate keys generates a new DID and:

1. Issues a signed `KeyRotationStatement` from the **old** DID, naming the new DID and a `validFrom` timestamp.
2. Re-presents itself to peers under the new DID. Peers that hold the rotation statement MAY auto-update their contact entry.
3. Re-issues credentials from each SCA against the new DID (see [RFC-0004 §Re-issuance](./0004-sca.md#re-issuance)).

`KeyRotationStatement` is a JWT signed by the old key:

```json
{
  "iss": "did:key:z6Mk...OLD",
  "sub": "did:key:z6Mk...NEW",
  "purpose": "key-rotation",
  "validFrom": "2026-09-01T00:00:00Z",
  "iat": 1756684800
}
```

Lost-key recovery is out of scope for v0.1 — losing the private key means losing the identity. Subsequent RFCs MAY define social or hardware-anchored recovery.

## Resolution

- `did:key` resolves locally (parse the multibase tail).
- `did:web` resolves over HTTPS; results SHOULD be cached for the `Cache-Control` window, default 1 hour if absent.

## Forbidden DID document fields (v0.1)

To keep verifier complexity bounded, a v0.1 DID document MUST contain only:

- `id`
- `verificationMethod` (Ed25519 only)
- `authentication`
- `assertionMethod`

Any other field MUST be ignored by v0.1 verifiers.

## Open questions

- Whether to additionally support `did:dht` or another lightweight registry method for individuals who want a stable DID without a domain.
- Whether `KeyRotationStatement` should be an actual W3C VC (delaying RFC-0003 dependency) or stay JWT-only.
