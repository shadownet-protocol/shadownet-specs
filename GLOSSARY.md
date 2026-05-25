# Glossary

| Term | Definition |
| --- | --- |
| A2A | [Google's Agent-to-Agent protocol](https://google.github.io/A2A/). Shadownet defines a profile. |
| AffiliationCredential | A VC asserting that a Subject is affiliated with an organization. Issued by a `did:web` org or an SCA the org controls. See [RFC-0003 §AffiliationCredential](./rfcs/0003-credentials.md#affiliationcredential). |
| Contact profile | Local-only metadata (notes, priority, scope, expiry) the Subject attaches to a contact; never transmitted. See [RFC-0007 §Contact profile](./rfcs/0007-mcp-tools.md#contact-profile). |
| CSR | Certificate Signing Request. JWT signed by the Subject; submitted to an SCA's `/issuance` to request a credential. |
| DID | [Decentralized Identifier](https://www.w3.org/TR/did-core/). The stable identity for a Shadow. |
| Freshness proof | Short-lived attestation that a long-lived credential has not been revoked. |
| Gateway | A rules-only filtering tier that fronts a backend Sidecar at the SNS endpoint. MX-equivalent. See [RFC-0005 §Gateway pattern](./rfcs/0005-sns.md#gateway-pattern). |
| Holder | The Shadow that holds and presents credentials about its subject. |
| Hub | (Later.) Themed stranger-matching server. Out of v0.1. |
| Institutional trust | Trust derived from a credential issuer — an SCA in the trust store, or a `did:web` org via domain control. See [RFC-0001 §Trust models](./rfcs/0001-overview.md#trust-models). |
| Invitation | An inbound envelope from an unknown sender carrying `hints.purpose: "invitation"`. Routed to quarantine; never invokes the host agent's LLM. See [RFC-0006 §Invitation envelopes](./rfcs/0006-a2a-profile.md#invitation-envelopes). |
| Issuer | An SCA, or (for AffiliationCredentials) an organization, that signs credentials. |
| Level | Sybil-resistance class of a SubjectCredential (e.g. `L1` email, `L2` ID document, `L3` unique person, `O1` verified org). Not a trust ranking. |
| MCP | [Model Context Protocol](https://modelcontextprotocol.io). The host-agent ⇄ Sidecar surface. |
| Predicate | JSON expression describing what a verifier requires of a presentation (see [RFC-0004 §Required-level predicates](./rfcs/0004-sca.md#required-level-predicates)). |
| Proof session | An on-protocol exchange between a Subject and an SCA used to prove eligibility for a credential level (see [RFC-0004 §Issuance flow](./rfcs/0004-sca.md#issuance-flow)). |
| Quarantine | The holding area at the receiver (commonly the gateway) for inbound from unknown senders. User-reviewed; cost-bounded by construction. See [RFC-0006 §Routing and quarantine](./rfcs/0006-a2a-profile.md#routing-and-quarantine). |
| Relational trust | Trust derived from the local contact graph and per-contact grants/profile, not from any issuer. See [RFC-0001 §Trust models](./rfcs/0001-overview.md#trust-models). |
| SCA | Shadow Certificate Authority. Issues credentials about a Shadow's subject. |
| Shadow | Personal AI agent acting on behalf of one human (or one organization). |
| Shadowname | Human-readable address for a Shadow (e.g. `mahdi@sh4dow.org`). |
| Sidecar | Local process that handles networking, keys, and storage for a Shadow. |
| SNS | Shadow Name Service. Resolves a Shadowname to an endpoint and a public key. |
| Subject | The entity a credential is about — a human or an organization. |
| SubjectCredential | A VC asserting a Sybil-resistance level about a Subject. Issued by an SCA. See [RFC-0003 §SubjectCredential — JWT shape](./rfcs/0003-credentials.md#subjectcredential--jwt-shape). |
| Trust store | A verifier's local policy — issuer trust (which SCAs at which levels) plus institutional trust (which orgs' affiliation claims). See [RFC-0004 §Trust store](./rfcs/0004-sca.md#trust-store). |
| VC | [Verifiable Credential](https://www.w3.org/TR/vc-data-model/). What an SCA or an organization issues. |
| Verifier | The party (peer Shadow, Hub, service) that checks a presented credential. |

Add new terms alphabetically.
