# Changelog

## [Unreleased]

- Initial v0.1 RFC set: 0001 Overview, 0002 Identity, 0003 Credentials, 0004 SCA, 0005 SNS, 0006 A2A Profile, 0007 MCP Tools.
- JSON Schemas for credential and A2A envelope.
- ROADMAP and wire-level Birthday-flow walkthrough (mixed deployment).
- RFC-0004: spec'd on-protocol proof flow (`/proof/start`, `/proof/status`, callbacks), `/freshness` request format, and required-level predicate grammar.
- RFC-0007: spec'd Sidecar→host-agent webhook contract (HMAC, replay window, retries) and `social_set_webhook` tool.
- DEVELOPMENT.md: full development plan — components, repos (Go/Python/TS SDKs + cloud + servers + conformance), language choices, cloud subdomain layout, third-party services, six-milestone roadmap, risks, open decisions.
- Decisions locked: GitHub org `shadownet-protocol`, MIT license across all repos (patent rights retained), Go 1.23+, no Postgres in `shadownet-go` (Store interface only).
- RFC-0004: clarified that proof `method` URIs are operator-defined; the protocol does not enumerate methods.
