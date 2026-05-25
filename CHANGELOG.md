# Changelog

## [Unreleased]

- RFC-0005: added §Lifecycle defining the record time-validity contract. Pins the wire invariant that resolution responses MUST NOT carry a pre-expired envelope, defines two valid provider implementation patterns (re-issuance on resolve vs. store-and-serve), gives normative client-side renewal guidance (re-register before `iat + ttl - max(60, ttl/10)`), and recommends resolvers distinguish expiry from malformedness in their error taxonomy. Recommended default `ttl` raised from the previous ad-hoc 300 s to 3600 s. Wire shape unchanged.
- RFC-0007: removed `social_set_webhook` tool and webhook delivery path (Path 2). Inbound delivery is now MCP notifications (Path 1) or `social_inbox_wait` long-poll (Path 2, RECOMMENDED). The webhook HMAC/retry/URL-constraint sections, §Compatibility headers, and §Coexistence with webhooks are all removed. `social_inbox_wait` is now the RECOMMENDED default.
- RFC-0008: removed `webhook_secret` field from integration bundle and `webhook` capability flag. `inbox-wait` flag marked RECOMMENDED.
- RFC-0009: removed `social_set_webhook` from `mcp:tools.write` scope.
- Schemas: removed `webhook_secret` from `integration-bundle.schema.json`; removed webhook references from `social-inbox-wait-result.schema.json`.
- Examples: updated `birthday-flow.md` and `free-form-coordination.md` to use `social_inbox_wait` instead of webhook delivery.

- Initial v0.1 RFC set: 0001 Overview, 0002 Identity, 0003 Credentials, 0004 SCA, 0005 SNS, 0006 A2A Profile, 0007 MCP Tools.
- JSON Schemas for credential and A2A envelope.
- Wire-level Birthday-flow walkthrough (mixed deployment).
- RFC-0004: on-protocol proof flow (`/proof/start`, `/proof/status`, callbacks), `/freshness` request format, required-level predicate grammar.
- RFC-0004: clarified that proof `method` URIs are operator-defined; the protocol does not enumerate methods.
- RFC-0007: Sidecar→host-agent inbound notification contract (`social_inbox_wait`, MCP notifications).
- Canonical domain landed: `sh4dow.org`. All `shadownet.example` placeholders replaced across spec, schemas, and example.
- RFC-0009: new RFC defining the OAuth 2.1 authorization profile for the Sidecar's MCP endpoint. Strict superset of the MCP authorization specification — composes RFC 9728 (Protected Resource Metadata), RFC 8414 (AS Metadata), RFC 7636 (PKCE), RFC 8707 (Resource Indicators), and optionally RFC 7591 (DCR), RFC 7009 (Revocation), RFC 8628 (Device Grant), RFC 7662 (Introspection). Independent conformance class; coexists with RFC-0008 paste-based onboarding.
- RFC-0007 §Transport: amended to accept OAuth 2.1 access tokens (RFC-0009) as an alternative to RFC-0008 paste-based bearer tokens on the same MCP endpoint.
- RFC-0008: added `oauth-authorize` capability flag and the optional `protected_resource_metadata` bundle field.
- RFC-0006: `interaction` is now OPTIONAL. Default envelope form is free-form text (`payload.text` + optional `hints`); typed Interaction Profiles become an opt-in optimization for cases where structure prevents ambiguity. Schema updated; new `payload_invalid` error code added for callees that choose to enforce a known profile.
- RFC-0007: `social_send` `interaction` argument is now optional, mirroring RFC-0006. Payload guidance documents free-form vs. typed shapes.
- New example `examples/free-form-coordination.md`: companion walkthrough showing the default text-payload envelope (the Birthday flow remains the typed-path example).
- RFC-0008 (new): Sidecar Onboarding Surface. Defines the integration-bundle endpoint (`GET <base>/v1/account/me/integration-bundle`), the `shadownet://connect?...` URI scheme with inline and handoff forms, and content-negotiated per-host install pages at `<base>/connect/<host>`. Conformance is independent of RFC-0007 — a Sidecar MAY implement one without the other. The set of recognized host slugs lives as a non-normative companion at `examples/well-known-hosts.md`; adding a host does not require a spec change.
- RFC-0007: new required tool `social_inbox_wait` — long-poll for inbox events. `event_id` is opaque to clients; on empty timeout the cursor returns the server's current high-water mark (or `null` for tenants that have never had an event). The same event delivered via `social_inbox_wait` and Path 1 notifications MUST carry byte-identical `event_id` strings so receivers can dedupe across transports.
- RFC-0007: renamed Path 1 notification surface from the singular `shadownet/inbox` to the per-event namespace `notifications/shadownet/<event-name>` (`inbox.message`, `task.update`, `freshness.expired`, `presentation.failed`). This is a pre-v1.0 rename, not additive — Sidecars and host agents that implemented the old name must update. The receiver-idempotency requirement now keys on `event_id` rather than `messageId` so it generalizes to non-message events.
