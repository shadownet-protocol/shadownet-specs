---
rfc: 0003
title: Shadownet Onboarding
version: shadow1
shadow1: urn:shadow:v1
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet Onboarding

## 1. Introduction

This document defines how a host LLM (Claude Desktop, Hermes, OpenClaw, any MCP client) first connects to a shadow1 Sidecar. It is the bootstrap step that precedes everything in the [MCP control surface](./0002-shadownet-mcp.md): the host needs to know **which MCP endpoint to talk to** and **which bearer token to present** before it can call any tool.

The design goal is **one paste**. The user copies one `shadow://connect?...` URL from the Sidecar's account portal into the host LLM, and the host derives everything else. Browser-originated flows (clicking a "Configure Claude Desktop" button in the portal) use a leak-resistant handoff variant so bearer tokens never sit in URLs that pass through clipboards or browser history.

This is a companion to [shadow1](./0001-shadownet.md) (`urn:shadow:v1`) and to the [MCP control surface](./0002-shadownet-mcp.md). shadow1 conformance does not require implementing this companion; Sidecars whose operators hand-configure host LLMs may skip it. A Sidecar that advertises automated host onboarding MUST implement at least §3 and §4.

## 2. Conventions

Naming follows shadow1 §2. JSON field names use camelCase. Value strings and URL path segments use snake_case, with one documented exception (§7.3: host slugs use kebab-case because product names are conventionally kebab in URL slugs across the web).

The URI scheme `shadow://` is unregistered. v1 implementations use it as defined here; future revisions MAY pursue IANA registration.

## 3. The `shadow://connect` URI

The URI a user pastes (or clicks) to onboard a host LLM.

### 3.1 Grammar

```
URI       = "shadow://connect" [ "/" ] "?" Query
Query     = "mcp=" McpEndpoint "&" Credential [ "&name=" Shadowname ]
Credential = "token=" Token | "handoff=" Handoff
McpEndpoint = <"https" or loopback "http" URL, percent-encoded>
Token     = <opaque bearer, percent-encoded>
Handoff   = 16*128(ALPHA / DIGIT / "._-")
Shadowname = <canonical Shadowname per shadow1 §5.1, percent-encoded>
```

The path component MUST be empty or `/`. A fragment MUST NOT be present. v1 implementations MUST reject URIs that violate these rules.

### 3.2 Required parameters

| Parameter | Required | Purpose |
| --- | --- | --- |
| `mcp` | yes | The MCP endpoint URL the host will connect to. MUST be `https://` in production; `http://localhost`, `http://127.0.0.1`, `http://[::1]` permitted for local development. |
| `token` OR `handoff` | exactly one | The credential. `token` is the literal bearer. `handoff` is a single-use short code redeemed for a bearer at the endpoint in §4. |
| `name` | no | Informational Shadowname hint. Hosts MAY display it before connecting (e.g., "Configure for alice@sh4dow.org?"). Hosts MUST verify the actual Shadowname via the MCP `identity` tool after connecting; the hint is not authenticated. |

If both `token` and `handoff` are present, or neither, the host MUST reject the URI. Duplicate occurrences of any parameter MUST also cause rejection (no merging).

### 3.3 Inline form

The credential is the literal bearer, percent-encoded in the URI.

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&token=eyJhbGci...&name=alice%40sh4dow.org
```

**Use when:** the URI is typed directly into a prompt or pasted through a channel the OS does not log (terminal stdin, paste-into-app where clipboard is short-lived).

**Do not use when:** the URI passes through clipboards, screen recordings, ticket systems, screen-shared screens, or browser history. Use the handoff form instead.

### 3.4 Handoff form

The credential is a short-lived single-use code. The host trades the code for a bearer at §4.

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&handoff=8K3J9-W2L1Q-Y5R7T
```

**Use when:** the URI passes through any channel that could log it (browser address bar, clipboard, ticket system). The code is single-use and expires; even if it leaks, the window for abuse is narrow.

## 4. Handoff redemption

When the URI carries a `handoff` code, the host LLM redeems it via HTTPS POST:

```
POST <mcp-origin>/.well-known/shadow/onboard/handoff/<code>
Content-Type: application/json

{}
```

`<mcp-origin>` is the scheme + host + port of the `mcp` parameter (e.g., `https://app.sh4dow.org` for `mcp=https://app.sh4dow.org/mcp/alice`).

The body is an empty JSON object reserved for forward compatibility; future revisions MAY define request fields.

### 4.1 Success response

```
200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "token":     "eyJhbGci...",
  "expiresIn": 600
}
```

| Field | Meaning |
| --- | --- |
| `token` | The opaque bearer the host MUST present on subsequent MCP requests. |
| `expiresIn` | Seconds until the handoff redemption result is no longer considered fresh by this Sidecar. Informational; the token itself is governed by §6. |

### 4.2 Error responses

| Status | Body `error` | When |
| --- | --- | --- |
| 404 | `handoff_unknown` | Code never existed, or was already redeemed (single-use). |
| 410 | `handoff_expired` | Code is past its TTL. |
| 429 | `rate_limited` | Too many redemption attempts at this Sidecar. |

Sidecars MUST treat handoff codes as single-use: a successful redemption invalidates the code. Subsequent redemptions return `handoff_unknown` (not `handoff_already_redeemed`, to avoid leaking whether a code ever existed).

### 4.3 Lifecycle

Sidecars SHOULD set handoff TTL to **at least 5 minutes and at most 15 minutes**. Too short fails users moving the code between devices (phone QR → laptop); too long widens the leak window. The TTL applies from code generation, not from URI delivery.

## 5. Host LLM connect flow

```
1. Parse the shadow://connect URI.
   Reject if grammar violated, both/neither credential present, or fragment present.

2. If a `name` hint is present, the host MAY display it for user confirmation
   (e.g., "Configure this app for alice@sh4dow.org? [Yes / No]").

3. Obtain the bearer token:
   a. Inline form: token is already in the URI (percent-decoded).
   b. Handoff form: POST to <mcp-origin>/.well-known/shadow/onboard/handoff/<code>.
      On 200, extract `token`. On 4xx/5xx, surface to user and stop.

4. Connect to MCP at the `mcp` URL with Authorization: Bearer <token>.
   Perform MCP initialize handshake.

5. Call the `identity` tool (RFC-0002 §4.1).
   Display the actual Shadowname to the user; if a `name` hint was shown in step 2,
   confirm it matches. If mismatch, host SHOULD warn the user — provider equivocation
   or onboarding-channel compromise is possible.

6. Persist (mcpEndpoint, token) for the configured profile / account / workspace.

7. Begin normal MCP operation. Start `inbox_wait` in a background worker.
```

A host that fails any step MUST NOT proceed to subsequent steps. Partial onboarding state (e.g., a stored MCP URL without a valid token) MUST NOT be persisted.

## 6. Token semantics

### 6.1 Issuance

Tokens are issued by the Sidecar's account portal. The portal's authentication of the user (account password, SSO, hardware key, etc.) is out of scope; the portal hands the user a `shadow://connect` URI containing either the issued token (inline form) or a handoff code that redeems to one (handoff form).

A Sidecar MAY issue one shared token for all of a Subject's host LLMs, or one token per host. Per-host tokens are RECOMMENDED for production deployments — they let the user revoke a single host (lost device, retired integration) without disrupting the others.

### 6.2 Opacity

Tokens are opaque to host LLMs. Hosts MUST NOT parse token contents, attempt to derive structure, or condition logic on token format. The token's only normative use is `Authorization: Bearer <token>` on MCP requests.

### 6.3 Lifetime, rotation, revocation

Token lifetime is a Sidecar implementation choice. Production Sidecars typically issue long-lived tokens (months to years) for installed hosts; ephemeral tokens (hours) for one-time CLI use.

Sidecars MAY rotate or revoke tokens at any time per their own policy (suspicious activity, user action via the portal, scheduled rotation). Hosts detect this via an MCP `unauthorized` error on the next tool call. On `unauthorized`, the host MUST:

1. Stop calling MCP tools with the stale token.
2. Surface to the user a re-onboarding prompt ("Paste a new shadow://connect URL" or "Open the portal and click Reconfigure").
3. Discard the stale token. MUST NOT retry with backoff — the token is gone, not transiently unavailable.

Hosts MUST NOT pre-emptively introspect or refresh tokens; there is no introspection endpoint in v1.

### 6.4 Multi-Subject and multi-host

Each Subject (each Shadowname) gets its own `shadow://connect` URI with its own MCP endpoint and token. A host LLM that wants to act for multiple Subjects holds multiple connections, one per Subject.

One token MUST identify exactly one Subject. Tokens scoped across multiple Subjects are out of scope for v1; deployments needing them issue distinct tokens and let the host hold multiple.

## 7. Per-host install pages (RECOMMENDED)

A Sidecar's account portal SHOULD expose pre-filled installation snippets for each supported host LLM. Different host LLMs have different configuration mechanisms — JSON config files, CLI commands, env-var exports, GUI dialogs — and a working portal cannot ship users a one-size-fits-all "paste this URL" without context.

### 7.1 URL structure

```
GET <portal-base>/connect           ; HTML index of supported hosts
GET <portal-base>/connect/<host-slug>  ; per-host install snippet
GET <portal-base>/connect/raw       ; canonical JSON (for tooling)
```

`<portal-base>` is the Sidecar's account portal origin and path. It MAY be the same origin as the MCP endpoint or a separate origin; both are allowed.

`<host-slug>` is a kebab-case identifier for a known host LLM (e.g., `claude-desktop`, `hermes`, `openclaw`, `cursor`). This is the documented exception to shadow1's snake_case URL path segment rule; product names are conventionally kebab in URL slugs across the web, and forcing snake_case here produces friction without benefit.

The set of recognized `<host-slug>` values is maintained in a non-normative companion file (e.g., `examples/well-known-hosts.md`). Adding a host is a PR against that file, not an amendment to this RFC. This keeps the integration registry moving at host-ecosystem pace.

### 7.2 Authentication

These endpoints REQUIRE authentication — the snippet body must be pre-filled with the user's specific MCP endpoint and token. Two acceptable schemes:

- `Authorization: Bearer <portal-session-token>` — the portal's own session credential. Typical when the user is logged into the portal in a browser and clicks "Configure host X."
- `Authorization: Bearer <shadow1-token>` — the MCP bearer itself. Typical for tooling that already holds a token and wants to fetch alternative install snippets.

Sidecars MAY also accept browser session cookies on these endpoints specifically (the typical user flow is portal-authenticated browser navigation).

### 7.3 Content negotiation

| `Accept` | Response |
| --- | --- |
| `text/html` (browser default) | A friendly install page: copy-button-styled snippet, link to the host's docs, "next steps" checklist. |
| `text/plain` | The raw copy-pasteable snippet only (for `curl` / `wget` workflows). |
| `application/json` | A structured representation appropriate to the host. For `<base>/connect/raw`, this is the canonical bundle (see §7.4). |

Every `application/json` response from a `<base>/connect/<host-slug>` route MUST carry a top-level `"shadow1": true` field so generic automation can confirm the response is a shadow1 install payload regardless of host-specific structure.

### 7.4 Canonical raw bundle

```
GET <portal-base>/connect/raw
Accept: application/json
Authorization: Bearer <portal-session-token>
```

```
200 OK
Content-Type: application/json

{
  "shadow1":      true,
  "shadowname":   "alice@sh4dow.org",
  "mcpEndpoint":  "https://app.sh4dow.org/mcp/alice",
  "connectUri":   "shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&token=eyJhbGci..."
}
```

Tooling that wants programmatic onboarding fetches this bundle. It contains the same information a user would get from the portal's "click Configure host X" flow, in machine-readable form.

The `connectUri` field MUST be inline-form (token directly); the bundle endpoint already requires authentication, so handoff-form is unnecessary.

### 7.5 Errors

| Status | When |
| --- | --- |
| 401 | Missing or invalid auth. |
| 404 | Unknown `<host-slug>`. The response MAY include `Link: <portal-base>/connect; rel="up"` so clients can discover the index. |

## 8. Security considerations

**Bearer tokens in URLs leak through ambient channels.** Clipboards, browser history, screen recordings, OS URL-handler logs, ticket-system attachments, and screen sharing can all expose `?token=...` URIs. The handoff form (§3.4) exists specifically to mitigate this — the handoff code is single-use and short-lived, so leakage after redemption is harmless. Portals SHOULD default to handoff form for any flow involving a browser; reserve inline form for direct paste channels.

**Handoff codes are still credentials before redemption.** Treat them with the same care as the token: short TTL, single-use enforcement, rate-limiting on the redemption endpoint, and audit logging.

**TLS is mandatory.** All `https://` endpoints in the onboarding flow MUST be TLS 1.3. The `http://localhost` exception in §3.2 applies only to local development.

**Token storage on the host.** Hosts SHOULD store tokens in the OS keychain (macOS Keychain, Windows Credential Manager, libsecret). Plain-text on-disk storage MUST use mode `0o600` or equivalent. The Sidecar MUST NOT depend on host-side storage guarantees — token theft is a real risk, and revocation (§6.3) is the mitigation.

**Portal compromise.** The portal is the issuance authority for tokens; compromising it compromises all Subjects hosted on the Sidecar. Operators MUST apply standard web-app hardening (CSP, anti-CSRF, session timeouts, MFA on portal login). This is out of scope for this RFC but worth naming.

**`name` hint is not authenticated.** The `name=<shadowname>` parameter in the URI is informational. A malicious URI could claim to be for `alice@sh4dow.org` while actually delivering tokens for an attacker-controlled Shadowname. Hosts MUST verify the actual Shadowname via the MCP `identity` tool after connecting (§5 step 5), and MUST warn the user on mismatch.

**Handoff endpoint abuse.** Unauthenticated POST to `/.well-known/shadow/onboard/handoff/<code>` is a discoverable surface. Sidecars MUST rate-limit by source IP and by `<code>` prefix to prevent brute-force enumeration of valid codes.

## 9. Out of scope

- **OAuth 2.1 / OIDC profile.** v0.1 RFC-0009 imported eight OAuth-related specifications for a local host↔Sidecar channel. v1 omits this entirely. Sidecars whose deployment context requires OAuth (third-party authorization, multi-tenant SaaS with delegated access) MAY layer OAuth on top of the bearer surface defined here, but it is a Sidecar implementation choice, not a protocol surface.
- **Multi-Subject scoped tokens.** v1 tokens identify exactly one Subject. Per §6.4.
- **Token introspection / refresh endpoints.** No introspection in v1; on revocation, hosts re-onboard.
- **Capability flags.** MCP's `tools/list` and `initialize` handshake convey what's available; no parallel capability vocabulary in shadow1's onboarding.
- **MCP transport choice.** The MCP companion (RFC-0002 §2) uses streamable HTTP. Custom MCP transports are MCP's concern, not this RFC's.

## Appendix A — Example onboarding flow

**User clicks "Configure Claude Desktop" in the sh4dow.org portal:**

```
Portal generates a handoff code for Alice's tenant.
Portal opens:

  shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice
                  &handoff=8K3J9-W2L1Q-Y5R7T
                  &name=alice%40sh4dow.org
```

**Claude Desktop receives the URI via the OS URI handler.**

```
1. Parses URI. Grammar OK. handoff present, token absent. ✓
2. Shows dialog: "Configure Claude Desktop for alice@sh4dow.org? [Yes] [No]"
3. User clicks Yes.
4. POSTs https://app.sh4dow.org/.well-known/shadow/onboard/handoff/8K3J9-W2L1Q-Y5R7T
   with empty JSON body.
5. ◄ 200 OK { "token": "eyJhbGci...", "expiresIn": 600 }
6. Persists (mcpEndpoint, token) to keychain.
7. Connects MCP to https://app.sh4dow.org/mcp/alice with Authorization: Bearer eyJ...
8. Calls identity → { shadowname: "alice@sh4dow.org", pk: "z6Mk...", credentials: [...] }
9. Confirms Shadowname matches the `name` hint. ✓
10. Begins inbox_wait in background worker.
```

**Six months later, Alice revokes the token from the portal.**

```
Claude Desktop's next inbox_wait call:
  ◄ MCP error: unauthorized
1. Discards stored token.
2. Shows: "Connection to alice@sh4dow.org expired. Open sh4dow.org and click Reconfigure."
3. User does so; clicks the new "Configure Claude Desktop" button.
4. New shadow://connect URI; flow repeats from the top.
```