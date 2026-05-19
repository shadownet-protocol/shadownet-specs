---
rfc: 0008
title: Sidecar Onboarding Surface
status: 📝 Draft
authors: []
created: 2026-05-11
---

# RFC 0008: Sidecar Onboarding Surface

## Summary

Defines the surface a host agent uses to *first connect* to a Sidecar: a single account-token-authenticated REST endpoint that returns everything the host agent needs (the integration bundle), a `shadownet://connect` URI scheme that encodes "where + with what credential" in one copy-pasteable string, and a per-host install-snippet endpoint that lets the Sidecar serve content-negotiated installation instructions pre-filled with the Subject's values.

This is a *local* onboarding surface — between the Subject's chosen host agent and its Sidecar. The MCP control surface used during an active session is in [RFC-0007](./0007-mcp-tools.md); the over-the-wire A2A profile is in [RFC-0006](./0006-a2a-profile.md). RFC-0008 governs only the bootstrap step.

## Motivation

Before this RFC, each host agent integration required users to copy-paste two or three separate values (token, base URL, optionally a webhook secret) from the Sidecar's account portal into the host agent's config file or environment. Every plugin reinvented the prompts. New host agents required prose-only install paragraphs in multiple READMEs that drifted as values changed.

The goal is one paste: the user copies a `shadownet://connect?...` URL once, and any compliant host agent can derive the rest.

## Conformance class

RFC-0008 defines a conformance class **independent** of RFC-0007. A Sidecar MAY conform to RFC-0007 (MCP control surface) without conforming to RFC-0008 (onboarding surface) — for instance, a self-hosted Sidecar that expects its operator to hand-configure the host agent has no need for one-token install. Conformance tests for the two RFCs MUST be runnable independently.

A Sidecar advertising the `bundle` capability flag (below) MUST implement all three surfaces in this RFC (bundle endpoint, connect URL parsing on the server side, per-host install pages).

## Design

### Integration bundle endpoint

```
GET <base>/v1/account/me/integration-bundle
Authorization: Bearer <account-token>
Accept: application/json
```

The token already identifies the Subject's tenant; no path parameter is required. Sidecars serving multiple tenants per token are out of scope for v0.1.

#### Response (200 OK)

The `did` field follows the DID-method rules in [RFC-0002](./0002-identity.md): individuals MUST be served `did:key`; organizations MUST be served `did:web`. Serving `did:web` for an individual or `did:key` for an organization is a protocol violation. Two representative shapes:

**Individual Subject** (`did:key`):

```json
{
  "shadownet:v": "0.1",
  "did": "did:key:z6MkSubjectPubkey...",
  "shadowname": "alice@app.sh4dow.org",
  "mcp_endpoint": "https://app.sh4dow.org/u/alice/mcp",
  "webhook_secret": "wh_eyJhbGciOi...",
  "supported_features": ["mcp", "webhook", "inbox-wait", "bundle", "connect-url"],
  "tool_names": ["social_send", "social_inbox", "social_inbox_wait", "social_set_webhook", "..."],
  "event_names": ["inbox.message", "task.update", "freshness.expired", "presentation.failed"],
  "version": "0.3.0"
}
```

**Organizational Subject** (`did:web`):

```json
{
  "shadownet:v": "0.1",
  "did": "did:web:acme.sh4dow.org",
  "shadowname": "events@acme.sh4dow.org",
  "mcp_endpoint": "https://acme.sh4dow.org/u/events/mcp",
  "webhook_secret": null,
  "supported_features": ["mcp", "webhook", "bundle", "connect-url"],
  "tool_names": ["social_send", "social_inbox", "social_set_webhook", "..."],
  "event_names": ["inbox.message", "task.update", "freshness.expired", "presentation.failed"],
  "version": "0.3.0"
}
```

#### Field semantics

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `shadownet:v` | string literal `"0.1"` | yes | Protocol version. Clients reject unknown values. |
| `did` | string | yes | Tenant DID; `did:key` for individuals, `did:web` for organizations (RFC-0002). |
| `shadowname` | string | yes | `local@provider` form per [RFC-0005](./0005-sns.md). |
| `mcp_endpoint` | string (https URL) | yes | The MCP streamable-HTTP endpoint per RFC-0007. |
| `webhook_secret` | string \| null | yes | Pre-provisioned HMAC secret for webhook deliveries; `null` if no webhook subscriber is registered. The host agent MAY use this when calling `social_set_webhook`, or generate its own. |
| `supported_features` | string array | yes | Capability flags — clients gate behavior on these. See [Capability flags](#capability-flags) below. |
| `tool_names` | string array | yes | The MCP tools this Sidecar exposes. Lets host agents pre-validate skill compatibility before connecting. |
| `event_names` | string array | yes | The event names this Sidecar may emit (via webhook, `social_inbox_wait`, or MCP notifications). |
| `version` | string | yes | Sidecar implementation version (free-form; semver RECOMMENDED). Not the protocol version. |

#### Capability flags

The normative set at v0.1, extensible:

| Flag | Meaning |
| --- | --- |
| `mcp` | The MCP endpoint at `mcp_endpoint` is live (RFC-0007). Required for any RFC-0007-compliant Sidecar. |
| `webhook` | Webhook delivery is supported (`social_set_webhook` works). |
| `bundle` | This endpoint is implemented (echoed for reflection). REQUIRED on RFC-0008-compliant Sidecars. |
| `connect-url` | Sidecar accepts `shadownet://connect` URLs (§ Connect URL scheme). |
| `inbox-wait` | The `social_inbox_wait` MCP tool is implemented ([RFC-0007 § social_inbox_wait](./0007-mcp-tools.md#social_inbox_wait)). |
| `mcp-notifications` | Sidecar emits the `notifications/shadownet/*` namespace on the MCP channel ([RFC-0007 § Path 1](./0007-mcp-tools.md#path-1-mcp-server-initiated-notification-in-band)). |
| `oauth-authorize` | Sidecar implements the OAuth 2.1 authorization profile in [RFC-0009](./0009-authorization.md). When set, the bundle MUST also include the `protected_resource_metadata` field naming the RFC 9728 PRM URL for this tenant. |

Clients MUST tolerate unknown future flags (forward compatibility). A flag's absence means the Sidecar does not advertise that capability — clients MUST NOT call the corresponding surface.

#### Errors

| Status | Body | When |
| --- | --- | --- |
| 401 | `{"error": "invalid_token"}` | Token missing, expired, or revoked. |
| 403 | `{"error": "forbidden"}` | Token valid but lacks the scope needed to read account state. |
| 404 | `{"error": "not_implemented"}` | Sidecar predates RFC-0008. Clients MUST treat this as "fall back to manual configuration" — not a fatal error. |

### Connect URL scheme

A single URI any host agent can parse to derive `(base, token)`.

#### Grammar

```
URI       = "shadownet://connect" [ "/" ] "?" Query
Query     = "base=" Base ( "&token=" Token | "&handoff=" Handoff )
Base      = <https-or-loopback-http URL, percent-encoded as a query value>
Token     = <opaque bearer token, percent-encoded>
Handoff   = <opaque short-code, [A-Za-z0-9._~-]{16,128}>
```

The path component MUST be empty or `/`. A fragment MUST NOT be present. Future revisions MAY define path segments; v0.1 clients MUST reject anything else.

#### Forms

**Inline** — token embedded directly. Suitable for paste flows the OS does not log (typed into a prompt, not screenshotted).

```
shadownet://connect?base=https://app.sh4dow.org&token=eyJhbGciOiJI...
```

**Handoff** — short-code that the host agent trades for the real token via a one-shot HTTP call. Use this when the URL may pass through clipboards, screen-shares, ticket systems, or similar contexts where the bearer should not be carried in clear text.

```
shadownet://connect?base=https://app.sh4dow.org&handoff=8K3J9-W2L1Q-Y5R7T
```

The host agent resolves the handoff via:

```
POST <base>/v1/account/connect/handoff/<handoff-code>
Content-Type: application/json
{}

→ 200 OK
{
  "shadownet:v": "0.1",
  "token": "eyJhbGci...",
  "expires_in": 600
}
```

Handoff codes are single-use and time-limited. RECOMMENDED TTL is 15 minutes — short enough to bound the window of leak, long enough that a user who generates the URL on their phone and copies it to their laptop has time. After redemption (or expiry) the code MUST return 404.

##### Reserved field

The field name `client_nonce` is reserved on the handoff request body. v0.1 clients MUST NOT send `client_nonce`; v0.1 servers MUST ignore it if present. Future RFC versions MAY assign semantics to this field — the most likely shape is a signed-echo variant where the server cryptographically attests the redemption back to the client — and the name is reserved against unrelated reuse to keep that path open. Implementations relying on `client_nonce` for replay defense today get no security benefit; the server-side single-use property is the only normative guarantee at v0.1.

#### Constraints

- Exactly one of `token` / `handoff` MUST be present. Both → reject. Neither → reject.
- `base` MUST use `https://` in production. `http://` is valid only for hosts in the loopback set (`localhost`, `127.0.0.1`, `::1`), matching the webhook URL allowlist in [RFC-0007 § URL constraints](./0007-mcp-tools.md#url-constraints).
- Multiple `base`, `token`, or `handoff` parameters → reject (no merging).

### Per-host install pages

A Sidecar advertising `bundle` MUST serve content-negotiated install pages per host.

#### Endpoints

| Path | Purpose |
| --- | --- |
| `GET <base>/connect` | HTML index listing supported hosts. |
| `GET <base>/connect/<host>` | Host-specific install snippet, content-negotiated. |
| `GET <base>/connect/raw` | The canonical bundle JSON (identical body to the `/v1/account/me/integration-bundle` response). |

The `<host>` slug is drawn from the registry maintained at [`examples/well-known-hosts.md`](../examples/well-known-hosts.md). That registry is non-normative — adding a host is a PR to the companion file, **not** an amendment to this RFC. This keeps spec churn out of the new-integration lifecycle.

#### Authentication

These endpoints REQUIRE `Authorization: Bearer <account-token>` (so the snippet body contains pre-filled values). Sidecars MAY also accept browser session cookies on these endpoints specifically, since the typical user flow is: account portal signs the user in, then redirects them to `<base>/connect/<host>`.

#### Content negotiation

| `Accept` header | Response |
| --- | --- |
| `text/html` (default for browsers) | A friendly install page: copy-button-styled snippet, link to the host's docs, "next steps" checklist. |
| `text/plain` | The raw copy-pasteable snippet only — suitable for `curl`/`wget` flows. |
| `application/json` | A structured representation appropriate to the host (e.g., `{ "mcpServerConfig": { ... } }` for hosts that configure MCP servers via JSON). For `<base>/connect/raw`, this is the canonical bundle. Sidecars SHOULD support `application/json` on every host route to enable automated bootstrap. |

Every `application/json` response from a `<base>/connect/<host>` route — including `raw` — MUST carry a top-level `"shadownet:v": "0.1"` field. This is the minimum contract generic automation can rely on to confirm the response is a Shadownet onboarding payload regardless of host. Host-specific fields sit alongside it; the canonical bundle at `raw` already satisfies the requirement via the bundle's own `shadownet:v`.

#### Snippet content per host

The companion registry specifies, for each registered slug, **what each host's snippet MUST contain** (e.g., the command incantation, env-var exports, JSON config block). The exact prose is implementation-specific. Snippets MUST be filled with the Subject's bundle values; values from any other tenant MUST NOT leak.

#### Errors

| Status | When |
| --- | --- |
| 401 | Missing or invalid auth. |
| 404 | Unknown `<host>` slug. The response MAY include `Link: <base>/connect; rel="up"` so clients can discover the index. |

## Alternatives

**Bake hosts into the RFC.** Considered and rejected. Each new integration target (Cursor, Continue, Aider, …) would require a spec amendment; the registry would be perpetually behind the implementations. The companion-file pattern lets the registry live where it can move fast.

**Use OAuth device flow instead of `shadownet://connect`.** Considered. Rejected for v0.1 because every Shadow already has an account token (the prerequisite for any Sidecar operation), and OAuth device flow would require standing up a second authorization server purely for first-connect ergonomics. The handoff form recovers most of the device-flow benefit (no bearer in copyable URLs) without the server-side weight.

**Embed `client_nonce` now, define semantics later.** Considered. Rejected: shipping a field that has no normative effect would mislead implementers into thinking it provides replay defense. Reserving the name keeps the option open without the false signal.

## Open questions

- **Handoff TTL.** 15 minutes is a reasonable default but has not been validated against an attacker model. Should the bundle advertise the TTL the server is using (`handoff_ttl_seconds`)?
- **Multi-tenant tokens.** v0.1 assumes one tenant per token (the `/me` path). A future version may need to support tokens scoped across multiple tenants; the `/me` shape leaves room for a sibling `/tenants/{id}` lookup but does not standardize it.
- **Snippet localization.** No mechanism today for non-English install pages. Probably belongs in a later RFC if the user base demands it.
