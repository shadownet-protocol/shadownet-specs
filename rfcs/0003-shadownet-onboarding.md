---
rfc: 0003
title: Shadownet Onboarding
version: 0.2
extension: urn:shadownet:0.2
status: 📝 Draft
authors: []
created: 2026-05-29
---

# Shadownet Onboarding

## 1. Introduction

This document defines how a host LLM (Claude Desktop, Hermes, OpenClaw, any MCP client) first connects to a Shadownet Sidecar. It is the bootstrap step that precedes everything in the [MCP control surface](./0002-shadownet-mcp.md): the host needs to know **which MCP endpoint to talk to** and **which bearer token to present** before it can call any tool.

The design goal is **one paste**. The user copies one `shadow://connect?...` URL from the Sidecar's account portal into the host LLM, and the host derives everything else. The same surface works whether the user is on the cloud reference deployment or running their own Sidecar from `docker compose`.

This is a companion to [Shadownet](./0001-shadownet.md) (`urn:shadownet:0.2`) and to the [MCP control surface](./0002-shadownet-mcp.md). The onboarding URI is the entry point regardless of which custody tier the user chose ([Shadownet §11](./0001-shadownet.md) defines self-hosted, hybrid BYO-key, and fully managed tiers); the bearer token the URI conveys identifies the Subject in all three. Shadownet conformance does not require implementing this companion.

## 2. Conventions

Naming follows [Shadownet §2](./0001-shadownet.md). JSON field names use camelCase. Value strings and URL path segments use snake_case. Host slugs (where they appear in URLs) use kebab-case as the documented exception.

The URI scheme `shadow://` is unregistered. v0.2 implementations use it as defined here; future revisions MAY pursue IANA registration.

## 3. The `shadow://connect` URI

The URI a user pastes (or clicks) to onboard a host LLM.

### 3.1 Grammar

```
URI         = "shadow://connect" [ "/" ] "?" Query
Query       = "mcp=" McpEndpoint "&" Credential
Credential  = "token=" AccessToken | "handoff=" Handoff
McpEndpoint = <"https" or loopback "http" URL, percent-encoded>
AccessToken = <opaque bearer, percent-encoded>
Handoff     = 16*128(ALPHA / DIGIT / "._-")
```

The path component MUST be empty or `/`. A fragment MUST NOT be present. v0.2 implementations MUST reject URIs that violate these rules.

### 3.2 Required parameters

| Parameter | Required | Purpose |
| --- | --- | --- |
| `mcp` | yes | The MCP endpoint URL the host will connect to. MUST be `https://` in production; `http://localhost`, `http://127.0.0.1`, `http://[::1]` permitted for local development. |
| `token` OR `handoff` | exactly one | The credential. `token` is the literal access token. `handoff` is a single-use short code redeemed for an access token at the endpoint in §4. |

The parameter is `mcp` (not `ep`) because [Shadownet §4.2](./0001-shadownet.md) already uses `ep=` as a DNS TXT key for the *provider's* HTTPS base URL — an unrelated slot. Two `ep` keys across the spec set with different meanings would recurringly confuse readers; `mcp` here is unambiguous.

If both `token` and `handoff` are present, or neither, the host MUST reject the URI. Duplicate occurrences of any parameter MUST also cause rejection (no merging).

### 3.3 Inline form

The credential is the literal access token, percent-encoded in the URI.

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&token=eyJhbGci...
```

**Use when:** the URI is typed directly into a prompt or pasted through a channel the OS does not log (terminal stdin, keyboard-entered into an app where the clipboard is not persisted).

**Do not use when:** the URI passes through clipboards, screen recordings, ticket systems, screen-shared screens, or browser history. Use the handoff form instead.

The token in an inline URI MAY be permanent (the simplest path for self-hosters) or short-lived (paired with the refresh mechanism in §7 — RECOMMENDED for production deployments that emit inline URIs through any channel that might log them). The Sidecar chooses; the wire shape is the same either way.

### 3.4 Handoff form

The credential is a short-lived single-use code. The host trades the code for an access token at §4.

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&handoff=8K3J9-W2L1Q-Y5R7T
```

**Use when:** the URI passes through any channel that could log it. The code is single-use and expires; even if it leaks, the window for abuse is narrow and the abuse leaves no usable token behind once the legitimate host redeems first.

## 4. Handoff redemption (OPTIONAL for inline-only Sidecars)

Sidecars that only ever emit inline-form URIs (typical for self-hosted / CLI flows) MAY skip this section entirely. Sidecars that emit handoff-form URIs MUST implement it.

When the URI carries a `handoff` code, the host LLM redeems it via HTTPS POST:

```
POST <mcp-origin>/.well-known/shadownet/onboard/handoff/<code>
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
  "accessToken":  "eyJhbGci...",
  "expiresAt":    "2026-05-31T10:00:00Z",
  "refreshToken": "rfRoT..."
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `accessToken` | yes | The opaque bearer the host MUST present on subsequent MCP requests. |
| `expiresAt` | no | ISO 8601 timestamp when the access token expires. Absent means the access token does not have a server-known expiry (typical for permanent tokens). Present means the host SHOULD refresh before this time (§7). |
| `refreshToken` | no | The refresh token, if the Sidecar offers renewal (§7). Absent means no renewal; on access-token expiry the host re-onboards via a new `shadow://connect` URI. |

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

2. Obtain the access token:
   a. Inline form: the token is in the URI (percent-decoded).
      No expiresAt is conveyed; host treats the access token as opaque and reacts
      to MCP `unauthorized` if/when the Sidecar revokes or rotates it.
   b. Handoff form: POST to <mcp-origin>/.well-known/shadownet/onboard/handoff/<code>.
      On 200, extract `accessToken` and optionally `expiresAt` and `refreshToken`.
      On 4xx/5xx, surface to user and stop.

3. Connect to MCP at the `mcp` URL with Authorization: Bearer <accessToken>.
   Perform MCP initialize handshake.

4. Call the `identity` tool (RFC-0002 §4.1).
   Display the actual Shadowname to the user as confirmation of which Subject
   this configuration is acting for.

5. Persist (mcpEndpoint, accessToken, refreshToken?, expiresAt?) for the
   configured profile / account / workspace.

6. Begin normal MCP operation. Start `inbox_wait` in a background worker.
```

A host that fails any step MUST NOT proceed to subsequent steps. Partial onboarding state (e.g., a stored MCP URL without a valid token) MUST NOT be persisted.

## 6. Token semantics

### 6.1 Issuance

Access tokens are issued by the Sidecar's account portal. The portal's own authentication of the user (account password, SSO, hardware key, OIDC, etc.) is out of scope; the portal hands the user a `shadow://connect` URI containing either the issued access token (inline) or a handoff code that redeems to one.

A Sidecar MAY issue one shared token for all of a Subject's host LLMs, or one token per host. Per-host tokens are RECOMMENDED for production deployments — they let the user revoke a single host without disrupting the others.

### 6.2 Opacity

Access tokens and refresh tokens are opaque to host LLMs. Hosts MUST NOT parse token contents, derive structure, or condition logic on token format. The only normative use of an access token is `Authorization: Bearer <token>` on MCP requests; the only normative use of a refresh token is at the endpoint in §7.2.

### 6.3 Lifetime

Token lifetime is a Sidecar implementation choice. Two production-relevant shapes:

- **Permanent access token, no refresh.** Simplest; ideal for trusted self-host scenarios (the user owns both the Sidecar and the host LLM). Revocation is the only kill switch. The inline URI form is fine when the URI travels through low-leak channels. For high-leak channels, use the handoff form even with permanent tokens — the handoff response is HTTPS POST and the access token never sits in a URL.
- **Short-lived access token (hours to days) + refresh token.** RECOMMENDED for production cloud deployments. The handoff response carries both. The host renews via §7 without prompting the user. Inline URIs in this shape carry only short-lived tokens; leakage of the inline URI exposes at most one short window.

Forcing all tokens to be permanent would be security-immature for production deployments; forcing all tokens to be short-lived would punish self-hosters who don't need the ceremony. v0.2 leaves the choice to the Sidecar and provides both surfaces.

### 6.4 Rotation and revocation

Sidecars MAY rotate or revoke tokens at any time per their own policy (suspicious activity, user action via the portal, scheduled rotation, refresh-token reuse detection per §7.3). Hosts detect revocation via an MCP `unauthorized` error on the next tool call.

On `unauthorized`:

1. If the host holds a refresh token (§7): attempt refresh. Success → continue with the new access token. Failure → fall to step 2.
2. Otherwise: discard the stale access token, surface a re-onboarding prompt to the user ("Paste a new `shadow://connect` URL" or "Open the portal and click Reconfigure"). MUST NOT retry the stale token.

Hosts MUST NOT pre-emptively introspect tokens; there is no introspection endpoint in v0.2.

### 6.5 Multi-Subject and multi-host

Each Subject (each Shadowname) gets its own `shadow://connect` URI with its own MCP endpoint and access token. A host LLM that wants to act for multiple Subjects holds multiple connections, one per Subject.

One access token MUST identify exactly one Subject. Tokens scoped across multiple Subjects are out of scope for v0.2; deployments needing them issue distinct tokens and let the host hold multiple.

## 7. Refresh tokens (OPTIONAL)

Sidecars MAY issue refresh tokens alongside access tokens so short-lived access tokens can be renewed without the user re-pasting a URI. This is the recommended pattern for production cloud deployments; self-hosters issuing permanent access tokens skip this section.

### 7.1 Issuance

Refresh tokens are returned only in the handoff redemption response (§4.1). Refresh tokens **MUST NOT** appear in `shadow://connect` URIs — the URI surface is for one-time delivery only, and adding a second long-lived credential to URLs doubles the leak surface for no operational benefit.

The presence of `refreshToken` in the handoff response signals that the Sidecar offers renewal. Its absence signals no renewal; on access-token expiry the host re-onboards per §6.4 step 2.

### 7.2 Refresh endpoint

```
POST <mcp-origin>/.well-known/shadownet/onboard/refresh
Authorization: Bearer <refresh-token>
Content-Type: application/json

{}
```

Response (`200 OK`, `Cache-Control: no-store`):

```json
{
  "accessToken":  "<new-access-token>",
  "expiresAt":    "2026-06-01T14:00:00Z",
  "refreshToken": "<new-refresh-token>"
}
```

Refresh tokens MUST rotate on every successful refresh: the response's `refreshToken` invalidates the one presented in the request. The host stores the new pair and discards the old.

### 7.3 Reuse detection

If a Sidecar receives a refresh request bearing a refresh token that has already been rotated (i.e., a refresh token presented after its replacement was issued), the Sidecar MUST treat it as a theft signal and **revoke the entire token family** — every access token and refresh token derived from this onboarding session. The host's next MCP call sees `unauthorized` and the user re-onboards.

This is the OAuth-standard pattern for refresh-token theft detection. It bounds the window of exposure when a refresh token leaks: either the legitimate host or the attacker will reach the refresh endpoint first; whichever loses sees its tokens revoked on the other's rotation.

### 7.4 Refresh errors

| Status | Body `error` | When |
| --- | --- | --- |
| 401 | `refresh_invalid` | Refresh token is unknown, already rotated, or revoked. The host MUST re-onboard. |
| 429 | `rate_limited` | Refresh requests exceeded the Sidecar's rate cap. |

## 8. Security considerations

**Bearer tokens in URLs leak through ambient channels.** Clipboards, browser history, screen recordings, OS URL-handler logs, ticket-system attachments, and screen sharing can all expose `?token=...` URIs. The handoff form (§3.4) exists specifically to mitigate this — the handoff code is single-use and short-lived, so leakage after redemption is harmless. Portals SHOULD default to handoff form for any flow involving a browser; reserve inline form for direct paste channels.

**Permanent access tokens compound the URL-leak risk.** If a Sidecar issues a permanent access token AND emits inline-form URIs, a single leak of one URI grants indefinite access. Production deployments SHOULD either (a) use handoff form so the access token never appears in a URI, or (b) pair short-lived access tokens with refresh tokens (§7) so an inline-URI leak exposes at most one short window. Self-hosters owning both ends MAY accept the permanent-inline tradeoff.

**Handoff codes are still credentials before redemption.** Treat them with the same care as access tokens: short TTL, single-use enforcement, rate-limiting on the redemption endpoint, audit logging.

**Refresh tokens are long-lived credentials.** They must not appear in URIs, must rotate on every use, and reuse MUST revoke the token family (§7.3). Hosts MUST store refresh tokens in the OS keychain (macOS Keychain, Windows Credential Manager, libsecret) or equivalent secure storage; plain-text on-disk storage of refresh tokens is a deployment bug.

**TLS is mandatory.** All `https://` endpoints in the onboarding flow MUST be TLS 1.3. The `http://localhost` exception in §3.2 applies only to local development.

**Token storage on the host.** Hosts SHOULD store both access and refresh tokens in the OS keychain. File-on-disk storage MUST use mode `0o600` or equivalent. The Sidecar MUST NOT depend on host-side storage guarantees — token theft is a real risk; revocation (§6.4) and refresh-token reuse detection (§7.3) are the mitigations.

**Custody-tier disclosure.** [Shadownet §11](./0001-shadownet.md) defines three custody tiers — self-hosted, hybrid BYO-key, and fully managed — with different trust models. The onboarding URI is the entry point regardless of tier; the tier itself is established at signup, not in the URI. Portals MUST disclose their custody tier at the moment the user generates an onboarding URI, so the user knows what they are about to install.

**Portal compromise.** The portal is the issuance authority for tokens; compromising it compromises all Subjects hosted on the Sidecar. Operators MUST apply standard web-app hardening (CSP, anti-CSRF, session timeouts, MFA on portal login). Out of scope for this RFC but worth naming.

**Handoff endpoint abuse.** Unauthenticated POST to `/.well-known/shadownet/onboard/handoff/<code>` is a discoverable surface. Sidecars MUST rate-limit by source IP and by `<code>` prefix to prevent brute-force enumeration of valid codes.

## 9. Out of scope

- **OAuth 2.1 / OIDC profile.** v0.1 RFC-0009 imported eight OAuth-related specifications for a local host↔Sidecar channel. v0.2 omits this entirely. The handoff redemption + refresh pattern defined here covers the cases that justify OAuth's authorization-code-with-PKCE flow, without the rest of OAuth's ceremony (DCR, PRM, AS metadata, scopes). A future companion `shadownet-oauth.md` MAY define a full OAuth 2.1 profile for SaaS-agent deployments where third-party authorization, scoped tokens, or standardized refresh semantics across multiple resource servers are required. v0.2 does not address that use case.
- **Multi-Subject scoped tokens.** v0.2 tokens identify exactly one Subject (§6.5).
- **Token introspection endpoints.** No introspection in v0.2; tokens are opaque and validated by use.
- **Capability flags.** MCP's `tools/list` and `initialize` handshake convey what's available; no parallel capability vocabulary in Shadownet's onboarding.
- **MCP transport choice.** The [MCP companion](./0002-shadownet-mcp.md) §2 uses streamable HTTP. Custom MCP transports are MCP's concern, not this RFC's.
- **Per-host installation pages.** Portals MAY expose pre-filled per-host install pages at URLs of their choosing. The protocol does not standardize this UX surface; doc writers refer users to "the portal's setup page" generically.

## Appendix A — Example onboarding flows

### A.1 Cloud handoff with refresh

**User clicks "Configure Claude Desktop" in the sh4dow.org portal:**

```
Portal generates a handoff code for Alice's tenant.
Portal opens:

  shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice
                  &handoff=8K3J9-W2L1Q-Y5R7T
```

**Claude Desktop receives the URI via the OS URI handler.**

```
1. Parses URI. Grammar OK. handoff present, token absent. ✓
2. POSTs https://app.sh4dow.org/.well-known/shadownet/onboard/handoff/8K3J9-W2L1Q-Y5R7T
   with empty JSON body.
3. ◄ 200 OK {
       "accessToken":  "eyJhbGci...",
       "expiresAt":    "2026-05-31T10:00:00Z",
       "refreshToken": "rfRoT..."
     }
4. Persists (mcpEndpoint, accessToken, expiresAt, refreshToken) to keychain.
5. Connects MCP to https://app.sh4dow.org/mcp/alice with Authorization: Bearer eyJ...
6. Calls identity → { shadowname: "alice@sh4dow.org", pk: "z6Mk..." }
7. Shows user: "Configured Claude Desktop for alice@sh4dow.org."
8. Begins inbox_wait in background worker.
```

**24 hours later, the access token approaches expiry.**

```
1. Claude Desktop's renewal worker sees expiresAt - now < 5 minutes.
2. POSTs https://app.sh4dow.org/.well-known/shadownet/onboard/refresh
   with Authorization: Bearer rfRoT...
3. ◄ 200 OK {
       "accessToken":  "eyJhbGci-new...",
       "expiresAt":    "2026-06-01T10:00:00Z",
       "refreshToken": "rfRoT-new..."
     }
4. Replaces stored (accessToken, expiresAt, refreshToken). Old refresh token discarded.
5. Continues operation seamlessly. User notices nothing.
```

**Six months later, Alice clicks "Revoke this host" in the portal.**

```
Claude Desktop's next refresh attempt:
  POST .../refresh with stale refresh token
  ◄ 401 { "error": "refresh_invalid" }

1. Surfaces: "Claude Desktop's connection to alice@sh4dow.org has been revoked.
   Open sh4dow.org and click Reconfigure."
2. User does so; new shadow://connect URI; flow repeats from the top.
```

### A.2 Self-hosted inline with permanent token

**Alice runs `docker compose up` on her self-hosted Sidecar. She gets the onboarding URI from her Sidecar's admin CLI:**

```
$ sidecar onboard --host claude-desktop
shadow://connect?mcp=http%3A%2F%2Flocalhost%3A7777%2Fmcp&token=alice-personal-permanent-token
```

**Alice pastes the URI into Claude Desktop's config dialog.**

```
1. Parses URI. Grammar OK. token present, handoff absent. ✓
2. Extracts token directly from URI (inline form). No expiresAt, no refreshToken.
3. Connects MCP to http://localhost:7777/mcp with Authorization: Bearer alice-...
4. Calls identity → { shadowname: "alice@aliceland.test", pk: "z6Mk..." }
5. Persists (mcpEndpoint, accessToken) to keychain.
6. Begins inbox_wait.
```

No refresh, no expiry. If Alice ever needs to revoke, she does it from the admin CLI; Claude Desktop sees `unauthorized` and prompts re-onboarding. For her single-user self-hosted setup this is sufficient and matches the simplicity she signed up for.