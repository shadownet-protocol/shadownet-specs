# Schemas

JSON Schemas for the wire artifacts defined by [`rfcs/0001-shadownet.md`](../rfcs/0001-shadownet.md) (Shadownet v0.2). All schemas use JSON Schema 2020-12.

These schemas are inputs to the conformance suite at [`shadownet-protocol/shadownet/conformance`](https://github.com/shadownet-protocol/shadownet/tree/main/conformance). The conformance suite at v0.2 references this repository directly.

| Schema | Artifact | Spec section |
| --- | --- | --- |
| [`credentials/credential.schema.json`](./credentials/credential.schema.json) | Credential JWT payload (`shadownet-cred+jwt`) | RFC 0001 §6.1 |
| [`credentials/csr.schema.json`](./credentials/csr.schema.json) | CSR JWT payload (`shadownet-csr+jwt`) | RFC 0001 §6.5 |
| [`messages/envelope.schema.json`](./messages/envelope.schema.json) | Envelope JWT payload (`shadownet-env+jwt`) | RFC 0001 §8.3 |
| [`agentcard/shadownet-extension.schema.json`](./agentcard/shadownet-extension.schema.json) | Shadownet extension fields on A2A AgentCard | RFC 0001 §5.3 |
| [`errors/problem.schema.json`](./errors/problem.schema.json) | Error response (RFC 7807 `application/problem+json`) | RFC 0001 §8.8 |

## Scope

Schemas in this directory cover the wire artifacts in [`rfcs/0001-shadownet.md`](../rfcs/0001-shadownet.md). Schemas for the MCP control surface ([`rfcs/0002-shadownet-mcp.md`](../rfcs/0002-shadownet-mcp.md)) and the onboarding URI ([`rfcs/0003-shadownet-onboarding.md`](../rfcs/0003-shadownet-onboarding.md)) live alongside those specs or with their reference implementations.

## What's not expressed in the schemas

Some normative constraints can't be expressed in JSON Schema 2020-12 alone and are checked at the conformance-suite level:

- **Envelope `exp - iat ≤ 300`.** Schema declares both as integers; the relationship is enforced by the suite per RFC 0001 §8.3.
- **Credential `exp - iat ≤ 2592000` (30 days for `org_affiliation`).** Same pattern. See RFC 0001 §6.3.
- **CSR `exp - iat ≤ 600`.** Recommended ceremony budget. See RFC 0001 §6.5.
- **Issuer authorization for `org`** per RFC 0001 §6.6 (same-domain / sub-domain / DNS delegate). Requires a DNS lookup; not a schema concern.
- **Signature verification** of any JWS. Cryptographic, not structural.
- **Replay defense** state. Receiver-side cache, not artifact structure.