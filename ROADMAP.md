# Roadmap

## Now

- **v0.2 spec drafted** — see [`rfcs/0001-shadownet.md`](./rfcs/0001-shadownet.md) (wire), [`rfcs/0002-shadownet-mcp.md`](./rfcs/0002-shadownet-mcp.md) (MCP control surface), [`rfcs/0003-shadownet-onboarding.md`](./rfcs/0003-shadownet-onboarding.md) (onboarding URI). v0.1 RFCs are preserved at the `v0.1-final` git tag.
- **Reference SDKs at v0.1.** Go SDK, Python SDK, conformance suite, and `shadownet-local` Sidecar currently ship the v0.1 surface and migrate to v0.2 as it stabilizes.

## v0.2 → 1.0

Goal: stabilize v0.2 against at least two interoperating implementations before declaring 1.0.

| Track | Action |
| --- | --- |
| Spec | Iterate on v0.2 against implementation feedback; revise as breaking changes are discovered. |
| Code | Migrate reference Sidecar and SDKs to v0.2 wire shapes (envelope, AgentCard, credential, CSR). |
| Code | Cross-implementation conformance suite against v0.2. |
| Code | Reference affiliation issuer for at least one canonical Hub use case (meetup or similar). |
| Code | Schemas (JSON Schema or equivalent) derived from v0.2 spec and shipped alongside the SDKs, not in this repo. |

## v0.3 candidates

| Candidate | Scope |
| --- | --- |
| **Keyed (domainless) addressing** | A second addressing path where identity is the Ed25519 public key, the endpoint is `key@ip:port` with a pinned TLS fingerprint, and no DNS or CA is required. Lets self-hosters skip "buy a domain, set DNS, get a cert." |
| **Provider-level store-and-forward** | A minimal MX-style relay at the provider so intermittent hosts (laptops, mobile devices) work without operating a gateway. Requires honest design of queue retention, abuse prevention, and gateway-to-backend pull semantics. |

## Later

| Track | Action |
| --- | --- |
| Spec | Intent profile RFCs (scheduling, intro, structured negotiation) — analogous to iCalendar over email. |
| Spec | AgentCard transparency log to mitigate provider equivocation. |
| Spec | Cross-Hub introductions and Hub-to-Hub federation patterns. |
| Code | Reference Hub server implementation. |