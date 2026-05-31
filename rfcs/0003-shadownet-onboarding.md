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

This document defines how a host LLM (Claude Desktop, Hermes, OpenClaw, any MCP client) first connects to a Shadownet Sidecar. The host must learn the MCP endpoint URL and the bearer token before it can invoke any tool defined in the [MCP control surface](./0002-shadownet-mcp.md).

The mechanism is a single `shadow://connect?...` URI the user pastes (or clicks) into the host LLM. The URI works for both cloud deployments and self-hosted Sidecars.

Companion to [Shadownet](./0001-shadownet.md) (`urn:shadownet:0.2`) and the [MCP control surface](./0002-shadownet-mcp.md). The bearer token conveyed by the URI identifies the Subject regardless of custody tier (see [Shadownet §11](./0001-shadownet.md)). Shadownet conformance does not require implementing this companion.

## 2. Conventions

Naming follows [Shadownet §2](./0001-shadownet.md). The URI scheme `shadow://` is unregistered.

## 3. The `shadow://connect` URI

### 3.1 Grammar

```
URI         = "shadow://connect" [ "/" ] "?" Query
Query       = "mcp=" McpEndpoint "&" Credential
Credential  = "token=" AccessToken | "handoff=" Handoff
McpEndpoint = <"https" or loopback "http" URL, percent-encoded>
AccessToken = <opaque bearer, percent-encoded>
Handoff     = 16*128(ALPHA / DIGIT / "._-")
```

The path component MUST be empty or `/`. A fragment MUST NOT be present. Implementations MUST reject URIs that violate these rules.

### 3.2 Parameters

| Parameter | Required | Purpose |
| --- | --- | --- |
| `mcp` | yes | The MCP endpoint URL the host will connect to. MUST be `https://` in production; `http://localhost`, `http://127.0.0.1`, `http://[::1]` permitted for local development. |
| `token` OR `handoff` | exactly one | The credential. `token` is the literal access token. `handoff` is a single-use short code redeemed for an access token at §4. |

If both `token` and `handoff` are present, or neither, the host MUST reject the URI. Duplicate occurrences of any parameter MUST also cause rejection.

### 3.3 Inline form

The access token is carried directly in the URI:

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&token=eyJhbGci...
```

**Use when** the URI is typed directly or pasted through a channel the OS does not log.

**Do not use when** the URI passes through clipboards, screen recordings, ticket systems, browser history, or screen sharing — use the handoff form instead.

The access token in an inline URI MAY be permanent or short-lived; the Sidecar chooses. Short-lived tokens SHOULD be paired with the refresh mechanism in §7.

### 3.4 Handoff form

The credential is a single-use short code:

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice&handoff=8K3J9-W2L1Q-Y5R7T
```

**Use when** the URI passes through any logging or shared channel. The code is single-use and short-lived.

## 4. Handoff redemption (OPTIONAL for inline-only Sidecars)

Sidecars that only emit inline-form URIs MAY skip this section. Sidecars that emit handoff-form URIs MUST implement it.

```
POST <mcp-origin>/.well-known/shadownet/onboard/handoff/<code>
Content-Type: application/json

{}
```

`<mcp-origin>` is the scheme + host + port of the `mcp` parameter. The body is an empty JSON object reserved for forward compatibility.

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
| `expiresAt` | no | ISO 8601 timestamp when the access token expires. Absent for permanent tokens; present means the host SHOULD refresh before this time (§7). |
| `refreshToken` | no | The refresh token, when the Sidecar offers renewal (§7). Absent means no renewal. |

### 4.2 Error responses

| Status | Body `error` | When |
| --- | --- | --- |
| 404 | `handoff_unknown` | Code never existed, or was already redeemed. |
| 410 | `handoff_expired` | Code is past its TTL. |
| 429 | `rate_limited` | Too many redemption attempts. |

Handoff codes MUST be single-use; a successful redemption invalidates the code. Subsequent redemptions return `handoff_unknown`.

### 4.3 Lifecycle

Sidecars SHOULD set handoff TTL to between 5 and 15 minutes, measured from code generation.

## 5. Host LLM connect flow

```
1. Parse the shadow://connect URI.
   Reject if grammar violated, both/neither credential present, or fragment present.

2. Obtain the access token:
   a. Inline form: extract from the URI (percent-decoded).
   b. Handoff form: POST to <mcp-origin>/.well-known/shadownet/onboard/handoff/<code>.
      On 200, extract `accessToken` and optionally `expiresAt` and `refreshToken`.
      On 4xx/5xx, surface to the user and stop.

3. Connect to MCP at the `mcp` URL with Authorization: Bearer <accessToken>.
   Perform MCP initialize handshake.

4. Call the `identity` tool (RFC-0002 §4.1) and display the Shadowname to the user.

5. Persist (mcpEndpoint, accessToken, refreshToken?, expiresAt?) in secure storage.

6. Begin normal MCP operation. Start `inbox_wait` in a background worker.
```

A host that fails any step MUST NOT proceed. Partial onboarding state MUST NOT be persisted.

## 6. Token semantics

### 6.1 Issuance

Access tokens are issued by the Sidecar's account portal. The portal's own user authentication (password, SSO, hardware key, OIDC, etc.) is out of scope.

A Sidecar MAY issue one shared token for all of a Subject's host LLMs, or one token per host. Per-host tokens are RECOMMENDED for production deployments.

### 6.2 Opacity

Access tokens and refresh tokens are opaque to host LLMs. Hosts MUST NOT parse token contents, derive structure, or condition logic on token format. The only normative use of an access token is `Authorization: Bearer <token>` on MCP requests; the only normative use of a refresh token is §7.2.

### 6.3 Lifetime

Token lifetime is a Sidecar implementation choice. Two production-relevant shapes:

- **Permanent access token, no refresh.** Revocation is the only kill switch. Suitable for trusted self-host scenarios.
- **Short-lived access token + refresh token.** The host renews via §7 without prompting the user. RECOMMENDED for production cloud deployments.

### 6.4 Rotation and revocation

Sidecars MAY rotate or revoke tokens at any time. Hosts detect this via an MCP `unauthorized` error.

On `unauthorized`:

1. If the host holds a refresh token (§7), attempt refresh. On success, continue with the new access token.
2. Otherwise, discard the stale access token and prompt the user to re-onboard. MUST NOT retry the stale token.

Hosts MUST NOT pre-emptively introspect tokens; there is no introspection endpoint.

### 6.5 Multi-Subject and multi-host

Each Subject gets its own `shadow://connect` URI with its own MCP endpoint and access token. One access token MUST identify exactly one Subject. Hosts acting for multiple Subjects hold multiple connections.

## 7. Refresh tokens (OPTIONAL)

Sidecars MAY issue refresh tokens alongside short-lived access tokens to enable renewal without user interaction. Self-hosters issuing permanent access tokens skip this section.

### 7.1 Issuance

Refresh tokens are returned only in the handoff redemption response (§4.1). Refresh tokens **MUST NOT** appear in `shadow://connect` URIs.

The presence of `refreshToken` in the handoff response signals that the Sidecar offers renewal. Absence signals no renewal.

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

Refresh tokens MUST rotate on every successful refresh. The response's `refreshToken` invalidates the one presented in the request. The host stores the new pair and discards the old.

### 7.3 Reuse detection

If a Sidecar receives a refresh request bearing a refresh token that has already been rotated, the Sidecar MUST revoke the entire token family — every access token and refresh token derived from this onboarding session.

### 7.4 Refresh errors

| Status | Body `error` | When |
| --- | --- | --- |
| 401 | `refresh_invalid` | Refresh token is unknown, already rotated, or revoked. The host MUST re-onboard. |
| 429 | `rate_limited` | Refresh requests exceeded the Sidecar's rate cap. |

## 8. Security considerations

**Bearer tokens in URLs leak through ambient channels** (clipboards, browser history, screen recordings, OS URL-handler logs, ticket-system attachments, screen sharing). Portals SHOULD default to handoff form for browser flows; reserve inline form for direct paste channels.

**Permanent access tokens compound URL-leak risk.** A Sidecar that issues permanent access tokens AND emits inline-form URIs exposes indefinite access on a single leak. Production deployments SHOULD use the handoff form, or pair short-lived access tokens with refresh tokens (§7).

**Handoff codes are credentials before redemption.** Apply the same care as access tokens: short TTL, single-use enforcement, rate-limiting on the redemption endpoint, audit logging.

**Refresh tokens are long-lived credentials.** They MUST NOT appear in URIs, MUST rotate on every use, and reuse MUST revoke the token family (§7.3). Hosts MUST store refresh tokens in the OS keychain (macOS Keychain, Windows Credential Manager, libsecret) or equivalent.

**TLS is mandatory.** All `https://` endpoints in the onboarding flow MUST be TLS 1.3. The `http://localhost` exception in §3.2 applies only to local development.

**Token storage on the host.** Hosts SHOULD store both access and refresh tokens in the OS keychain. File-on-disk storage MUST use mode `0o600` or equivalent.

**Custody-tier disclosure.** Portals MUST disclose their custody tier (see [Shadownet §11](./0001-shadownet.md)) at the moment the user generates an onboarding URI.

**Portal compromise.** The portal is the issuance authority for tokens; compromising it compromises all Subjects on the Sidecar. Operators MUST apply standard web-app hardening.

**Handoff endpoint abuse.** Sidecars MUST rate-limit the handoff redemption endpoint by source IP and by `<code>` prefix to prevent brute-force enumeration.

## 9. Out of scope

- **OAuth 2.1 / OIDC profile.** v0.2 does not define an OAuth profile. A future companion MAY define one for SaaS-agent deployments needing third-party authorization, scoped tokens, or standardized refresh semantics across multiple resource servers.
- **Multi-Subject scoped tokens.** v0.2 tokens identify exactly one Subject (§6.5).
- **Token introspection endpoints.** Tokens are opaque and validated by use.
- **Capability flags.** MCP's `tools/list` and `initialize` handshake convey what's available.
- **MCP transport choice.** See the [MCP companion](./0002-shadownet-mcp.md) §2.
- **Per-host installation pages.** Portals MAY expose pre-filled per-host install pages at URLs of their choosing. The protocol does not standardize this UX.

## Appendix A — Example onboarding flows

### A.1 Cloud handoff with refresh

The user clicks "Configure Claude Desktop" in the portal. The portal generates a handoff code and opens:

```
shadow://connect?mcp=https%3A%2F%2Fapp.sh4dow.org%2Fmcp%2Falice
                &handoff=8K3J9-W2L1Q-Y5R7T
```

Claude Desktop:

```
1. Parses URI. handoff present, token absent. ✓
2. POSTs https://app.sh4dow.org/.well-known/shadownet/onboard/handoff/8K3J9-W2L1Q-Y5R7T
3. ◄ 200 OK {
       "accessToken":  "eyJhbGci...",
       "expiresAt":    "2026-05-31T10:00:00Z",
       "refreshToken": "rfRoT..."
     }
4. Persists tokens to keychain.
5. Connects MCP to https://app.sh4dow.org/mcp/alice with Authorization: Bearer eyJ...
6. Calls identity → { shadowname: "alice@sh4dow.org", pk: "z6Mk..." }
7. Begins inbox_wait in background.
```

Before `expiresAt`, the renewal worker:

```
1. POSTs https://app.sh4dow.org/.well-known/shadownet/onboard/refresh
   with Authorization: Bearer rfRoT...
2. ◄ 200 OK {
       "accessToken":  "eyJhbGci-new...",
       "expiresAt":    "2026-06-01T10:00:00Z",
       "refreshToken": "rfRoT-new..."
     }
3. Replaces stored tokens. Continues without user prompt.
```

When the user revokes the host from the portal:

```
Next refresh attempt:
  POST .../refresh with stale refresh token
  ◄ 401 { "error": "refresh_invalid" }

Host surfaces a re-onboarding prompt.
```

### A.2 Self-hosted inline with permanent token

The user gets the URI from their Sidecar's admin CLI:

```
$ sidecar onboard --host claude-desktop
shadow://connect?mcp=http%3A%2F%2Flocalhost%3A7777%2Fmcp&token=alice-permanent
```

Claude Desktop:

```
1. Parses URI. token present, handoff absent. ✓
2. Extracts token from URI. No expiresAt, no refreshToken.
3. Connects MCP to http://localhost:7777/mcp with Authorization: Bearer alice-permanent
4. Calls identity → { shadowname: "alice@aliceland.test", pk: "z6Mk..." }
5. Persists token to keychain. Begins inbox_wait.
```

Revocation is via the admin CLI; the host sees `unauthorized` on the next MCP call and prompts for a new URI.