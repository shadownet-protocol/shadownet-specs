# Changelog

## [Unreleased]

- Initial v0.1 RFC set: 0001 Overview, 0002 Identity, 0003 Credentials, 0004 SCA, 0005 SNS, 0006 A2A Profile, 0007 MCP Tools.
- JSON Schemas for credential and A2A envelope.
- Wire-level Birthday-flow walkthrough (mixed deployment).
- RFC-0004: on-protocol proof flow (`/proof/start`, `/proof/status`, callbacks), `/freshness` request format, required-level predicate grammar.
- RFC-0004: clarified that proof `method` URIs are operator-defined; the protocol does not enumerate methods.
- RFC-0007: Sidecar→host-agent webhook contract (HMAC, replay window, retries) and `social_set_webhook` tool.
