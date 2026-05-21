---
rfc: 0001
title: Shadownet Protocol Overview
status: 📝 Draft
authors: []
created: 2026-05-02
---

# RFC 0001: Shadownet Protocol Overview

## Summary

Shadownet is a protocol for personal AI agents (Shadows) to discover each other, present cryptographic claims about who they are (or who they represent), and communicate over [Google A2A](https://google.github.io/A2A/). It is local-first by default and federated by design: no single SNS or SCA is privileged by the protocol.

This RFC fixes scope, vocabulary, actors, and the layer cake. Each layer is specified by a separate v0.1 RFC.

## Actors

- **Subject** — a human (`L1`–`L3`) or organization (`O1`). The entity a credential is *about*.
- **Shadow** — the agent acting on a Subject's behalf. Identified by a [DID](../GLOSSARY.md). One Subject may have one or more Shadows; a Shadow has exactly one Subject.
- **Sidecar** — local process that holds the Shadow's keys, contacts, history, and exposes [MCP](../GLOSSARY.md) tools to a host agent.
- **Issuer (SCA)** — signs Verifiable Credentials about Subjects.
- **Verifier** — anything that checks a presented credential. Most commonly another Shadow during an A2A handshake.
- **SNS provider** — operates a Shadowname → endpoint + key resolver.

## Layer cake

```
                          MCP tools             ← RFC-0007
                              │
                       Shadow agent logic
                              │
              ┌───────────────┴───────────────┐
              │       A2A profile             │ ← RFC-0006
              │  (handshake, VC presentation) │
              └───────────────┬───────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
          Credentials        SNS          Identity
          (RFC-0003)      (RFC-0005)      (RFC-0002)
              │
            SCA
          (RFC-0004)
```

Reading order for implementers: 0002 → 0003 → 0004 → 0005 → 0006 → 0007.

## v0.1 scope

In: known-contact coordination — two or more Shadows whose Subjects are already in each other's contact graphs negotiate over A2A on their humans' behalf. The canonical example is in [`examples/birthday-flow.md`](../examples/birthday-flow.md).

Out: stranger-matching (Hubs), interaction-content schemas (scheduling/intro/dating profiles), payment, reputation, federated SNS/SCA discovery. These are post-v0.1.

## Federation

The protocol does not name any SNS or SCA as canonical. Verifiers maintain a **trust store** that lists which SCAs they accept and at which assurance levels. The reference cloud deployment at v0.1 ships as the only entry in the default trust store, but is structurally one provider of many. See [RFC-0004 §Trust store](./0004-sca.md#trust-store).

## Conformance classes

| Class | Defined by | Required of |
| --- | --- | --- |
| Shadow | RFCs 0002, 0003, 0006 | Anything that holds an identity and speaks A2A as a Shadownet peer. |
| Sidecar | + RFC-0007 | The local process exposing MCP. May be the same code as the Shadow. |
| SCA | RFCs 0003, 0004 | An issuer of Shadownet credentials. |
| SNS provider | RFC-0005 | A name resolver. |

A single deployment may fulfill multiple classes (the v0.1 cloud service is SCA + SNS + a multi-tenant Sidecar host).

## Cryptography

Mandatory-to-implement:

- **Signatures**: Ed25519 (EdDSA per [RFC 8032](https://datatracker.ietf.org/doc/html/rfc8032)).
- **Hash**: SHA-256.
- **Transport**: TLS 1.3 ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).
- **Token format**: JWT ([RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)) with `EdDSA` `alg`.

Other algorithms MAY be negotiated; v0.1 verifiers MUST support the above.

## Versioning

Wire artifacts (credentials, A2A envelope, SNS records) carry a `shadownet:v` claim with semver-flavored value. v0.1.x changes are additive only; breaking changes bump the major.

## Reference implementation

[shadownet-local](https://github.com/shadownet-protocol/shadownet-local) implements the Sidecar today (single-tenant; raw-key identity + JWT-EdDSA, no DID/VC yet). Multi-tenant operation, DID/VC migration, and the cloud SCA/SNS service are v0.1 work items in [`ROADMAP.md`](../ROADMAP.md).
