# Glossary

Alphabetical. Terms are defined as they appear in [`rfcs/0001-shadownet.md`](./rfcs/0001-shadownet.md) (v0.2).

| Term | Definition |
| --- | --- |
| A2A | [Google's Agent-to-Agent protocol](https://a2a-protocol.org/). Shadownet ships as an A2A extension under `urn:shadownet:0.2`. |
| AgentCard | A2A's signed metadata document describing an agent. The provider issues one per Shadow at `<ep>/identity/<local>`, binding the Shadowname to a signing key. |
| Credential | A JWS-compact JWT signed by an issuer attesting `org_affiliation` for a Shadowname. See RFC 0001 §6. |
| CSR | A JWS-compact JWT a Subject sends to an issuer to request a credential. See RFC 0001 §6.5. |
| Envelope | The Shadownet message wrapper carried as a JWS-compact string in A2A `message.metadata["urn:shadownet:0.2"]`. Signed by the sender's key. See RFC 0001 §8. |
| Hub | An organization that admits members under a contextual vetting model (dating, hiring, meetup, etc.) and issues `org_affiliation` credentials to vetted members. The Shadownet Sybil-resistance mechanism. |
| Issuer | A party that signs credentials. Identified by a domain. See RFC 0001 §6.6 for delegation rules. |
| MCP | [Model Context Protocol](https://modelcontextprotocol.io). The host LLM ↔ Sidecar control surface (companion RFC 0002). |
| Provider | Authoritative for the Shadowname → AgentCard binding within a domain. Identified by domain; signs AgentCards with a key published in DNS at `_shadownet.<domain>`. |
| Receiver | A Shadow processing an inbound envelope. Applies validation (§8.6) then classification (§9). |
| Shadow | The addressable agent. Identified by a Shadowname. |
| Shadowname | Human-readable address of a Shadow, `local@provider` (e.g., `alice@sh4dow.org`). |
| Sidecar | The local process that holds a Shadow's keys, state, and speaks the wire. May serve one Subject (self-hosted) or many (multi-tenant). |
| Subject | The entity an identity is about — a human, an organization persona, or an automated service. |
| Trust store | A verifier's flat list of `(issuer-domain, [accepted-kinds])` tuples. See RFC 0001 §7.1. |

Add new terms alphabetically.