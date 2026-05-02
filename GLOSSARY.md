# Glossary

| Term | Definition |
| --- | --- |
| Shadow | Personal AI agent acting on behalf of one human (or one organization). |
| Sidecar | Local process that handles networking, keys, and storage for a Shadow. |
| Shadowname | Human-readable address for a Shadow (e.g. `mahdi@shadownet.example`). |
| SNS | Shadow Name Service. Resolves a Shadowname to an endpoint and a public key. |
| SCA | Shadow Certificate Authority. Issues credentials about a Shadow's subject. |
| A2A | [Google's Agent-to-Agent protocol](https://google.github.io/A2A/). Shadownet defines a profile. |
| MCP | [Model Context Protocol](https://modelcontextprotocol.io). The host-agent ⇄ Sidecar surface. |
| DID | [Decentralized Identifier](https://www.w3.org/TR/did-core/). The stable identity for a Shadow. |
| VC | [Verifiable Credential](https://www.w3.org/TR/vc-data-model/). What an SCA issues. |
| Trust store | The set of SCAs (and the levels at which) a verifier accepts. |
| Level | Assurance class of a credential (e.g. `L1` email, `L2` phone+ID, `L3` unique person, `O1` verified org). |
| Freshness proof | Short-lived attestation that a long-lived credential has not been revoked. |
| Subject | The entity a credential is about — a human or an organization. |
| Holder | The Shadow that holds and presents credentials about its subject. |
| Verifier | The party (peer Shadow, Hub, service) that checks a presented credential. |
| Issuer | An SCA that signs credentials. |
| Hub | (Later.) Themed stranger-matching server. Out of v0.1. |
| Predicate | JSON expression describing what a verifier requires of a presentation (see [RFC-0004 §Required-level predicates](./rfcs/0004-sca.md#required-level-predicates)). |
| Proof session | An on-protocol exchange between a Subject and an SCA used to prove eligibility for a credential level (see [RFC-0004 §Issuance flow](./rfcs/0004-sca.md#issuance-flow)). |
| CSR | Certificate Signing Request. JWT signed by the Subject; submitted to an SCA's `/issuance` to request a credential. |

Add new terms alphabetically.
