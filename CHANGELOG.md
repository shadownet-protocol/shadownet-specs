# Changelog

## [Unreleased]

- Initial v0.1 RFC set: 0001 Overview, 0002 Identity, 0003 Credentials, 0004 SCA, 0005 SNS, 0006 A2A Profile, 0007 MCP Tools.
- JSON Schemas for credential and A2A envelope.
- Wire-level Birthday-flow walkthrough (mixed deployment).
- RFC-0004: on-protocol proof flow (`/proof/start`, `/proof/status`, callbacks), `/freshness` request format, required-level predicate grammar.
- RFC-0004: clarified that proof `method` URIs are operator-defined; the protocol does not enumerate methods.
- RFC-0007: Sidecar→host-agent webhook contract (HMAC, replay window, retries) and `social_set_webhook` tool.
- Canonical domain landed: `sh4dow.org`. All `shadownet.example` placeholders replaced across spec, schemas, and example.
- RFC-0007: added §Compatibility headers under §Inbound notifications. Sidecars MAY emit additional HMAC headers in alternate formats (e.g. `X-Webhook-Signature`) for ecosystem interop; canonical headers remain mandatory; receivers using only a compatibility header MUST still verify `X-Shadownet-Sidecar-Ts` for replay defense (or explicitly accept the loss).
- RFC-0006: `interaction` is now OPTIONAL. Default envelope form is free-form text (`payload.text` + optional `hints`); typed Interaction Profiles become an opt-in optimization for cases where structure prevents ambiguity. Schema updated; new `payload_invalid` error code added for callees that choose to enforce a known profile.
- RFC-0007: `social_send` `interaction` argument is now optional, mirroring RFC-0006. Payload guidance documents free-form vs. typed shapes.
- New example `examples/free-form-coordination.md`: companion walkthrough showing the default text-payload envelope (the Birthday flow remains the typed-path example).
- RFC-0008 (new): Sidecar Onboarding Surface. Defines the integration-bundle endpoint (`GET <base>/v1/account/me/integration-bundle`), the `shadownet://connect?...` URI scheme with inline and handoff forms, and content-negotiated per-host install pages at `<base>/connect/<host>`. Conformance is independent of RFC-0007 — a Sidecar MAY implement one without the other. The set of recognized host slugs lives as a non-normative companion at `examples/well-known-hosts.md`; adding a host does not require a spec change.
- RFC-0007: new required tool `social_inbox_wait` — long-poll for inbox events, for host agents that cannot serve an inbound webhook. `event_id` is opaque to clients; on empty timeout the cursor returns the server's current high-water mark (or `null` for tenants that have never had an event). The same event delivered via webhook, `social_inbox_wait`, and Path 1 notifications MUST carry byte-identical `event_id` strings so receivers can dedupe across transports.
- RFC-0007: renamed Path 1 notification surface from the singular `shadownet/inbox` to the per-event namespace `notifications/shadownet/<event-name>` (`inbox.message`, `task.update`, `freshness.expired`, `presentation.failed`). This is a pre-v1.0 rename, not additive — Sidecars and host agents that implemented the old name must update. Webhook payloads gain a top-level `event_id` field; the receiver-idempotency requirement now keys on `event_id` rather than `messageId` so it generalizes to non-message events.
