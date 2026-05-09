# Changelog

## [Unreleased]

- Initial v0.1 RFC set: 0001 Overview, 0002 Identity, 0003 Credentials, 0004 SCA, 0005 SNS, 0006 A2A Profile, 0007 MCP Tools.
- JSON Schemas for credential and A2A envelope.
- Wire-level Birthday-flow walkthrough (mixed deployment).
- RFC-0004: on-protocol proof flow (`/proof/start`, `/proof/status`, callbacks), `/freshness` request format, required-level predicate grammar.
- RFC-0004: clarified that proof `method` URIs are operator-defined; the protocol does not enumerate methods.
- RFC-0007: Sidecar→host-agent webhook contract (HMAC, replay window, retries) and `social_set_webhook` tool.
- Canonical domain landed: `sh4dow.org`. All `shadownet.example` placeholders replaced across spec, schemas, and example.
- RFC-0006: `interaction` is now OPTIONAL. Default envelope form is free-form text (`payload.text` + optional `hints`); typed Interaction Profiles become an opt-in optimization for cases where structure prevents ambiguity. Schema updated; new `payload_invalid` error code added for callees that choose to enforce a known profile.
- RFC-0007: `social_send` `interaction` argument is now optional, mirroring RFC-0006. Payload guidance documents free-form vs. typed shapes.
- New example `examples/free-form-coordination.md`: companion walkthrough showing the default text-payload envelope (the Birthday flow remains the typed-path example).
