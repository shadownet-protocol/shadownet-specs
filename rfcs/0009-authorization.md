---
rfc: 0009
title: Sidecar Authorization (OAuth 2.1 Profile)
status: 📝 Draft
authors: []
created: 2026-05-10
---

# RFC 0009: Sidecar Authorization (OAuth 2.1 Profile)

## Summary

Defines an OAuth 2.1 authorization profile for the Sidecar's MCP endpoint, so OAuth-capable host agents (Claude Code, Cursor, Codex CLI, future hosts implementing the MCP authorization specification) can connect to a Sidecar using their built-in authorization machinery — no Shadownet-specific code in the host.

This RFC is **a strict superset of the MCP authorization specification** at [`/specification/latest/basic/authorization`](https://modelcontextprotocol.io/specification/latest/basic/authorization). A Sidecar conforming to RFC-0009 is automatically conformant with that MCP profile. Where this RFC adds anything beyond the MCP profile, the addition is labelled **Shadownet extension** and is OPTIONAL.

The paste-based onboarding surface in [RFC-0008](./0008-onboarding.md) remains the authoritative path for host agents that do not implement OAuth (Hermes Agent, SDK consumers, scripting). Both surfaces coexist; a Sidecar MAY advertise either, both, or neither.

## Conformance class

RFC-0009 defines a conformance class **independent** of RFC-0007 and RFC-0008. A Sidecar MAY implement RFC-0007 (MCP control surface) without implementing RFC-0009, and vice versa. Sidecars conforming to RFC-0009 MUST advertise the `oauth-authorize` capability flag in the RFC-0008 integration bundle (where RFC-0008 is also implemented) and in the OAuth 2.0 Protected Resource Metadata document (§ Discovery).

Conformance tests for RFC-0009 MUST be runnable against any HTTP MCP endpoint independent of whether the Sidecar implements RFC-0008.

## Standards compliance

This profile composes the following normative standards:

| Standard | Role |
| --- | --- |
| [OAuth 2.1 (draft-ietf-oauth-v2-1)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1) | Core framework. Mandates PKCE, drops implicit flow, drops password grant, requires exact redirect-URI matching. |
| [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728) | Protected Resource Metadata (PRM). The Sidecar publishes the PRM document; the host fetches it to discover which authorization server(s) to use. |
| [RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414) | OAuth 2.0 Authorization Server Metadata. The authorization server publishes its endpoint inventory. |
| [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) | PKCE. Required for the Authorization Code flow. |
| [RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707) | Resource Indicators for OAuth 2.0. Used to bind issued tokens to the Sidecar's MCP endpoint URL. |
| [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) | Dynamic Client Registration. Optional but RECOMMENDED. |
| [RFC 6749 § 6](https://datatracker.ietf.org/doc/html/rfc6749#section-6) | Refresh tokens. |
| [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) | Token Revocation. **Shadownet extension** — OPTIONAL. |
| [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) | Device Authorization Grant. **Shadownet extension** — OPTIONAL. |
| [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) | Token Introspection. **Shadownet extension** — internal AS choice. |

The MCP authorization specification is normative; where this RFC and the MCP specification differ, the MCP specification controls and this RFC is in error.

## Architecture

The Sidecar's MCP endpoint is an OAuth 2.1 **Resource Server**. It validates access tokens presented by host agents and serves MCP requests when validation succeeds.

The **Authorization Server** that issues those tokens MAY be:

- **Co-located** with the Sidecar (sidecar = AS, single process serves both). RECOMMENDED for v0.1 deployments — minimizes operational complexity and matches the self-sovereign Sidecar model.
- **Separate**, addressed by a distinct origin (e.g. a dedicated `auth.example.org` identity provider). The Sidecar's PRM document lists the AS origin; the host agent fetches AS metadata from that origin.

The PRM document MAY list multiple authorization servers; the host agent selects one according to its policy. The protocol is identical in either deployment shape; this RFC does not constrain the choice.

## Discovery

### Unauthenticated MCP request response

Any MCP request without a valid access token MUST receive `401 Unauthorized` with a `WWW-Authenticate` header whose `resource_metadata` parameter points at the Sidecar's PRM document:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="mcp",
  resource_metadata="https://sidecar.example.org/u/<shadowname>/.well-known/oauth-protected-resource"
```

`realm="mcp"` is REQUIRED. Additional `error` and `error_description` parameters MAY be included per OAuth 2.1 § 5.3.

### Protected Resource Metadata (RFC 9728)

```
GET <mcp-endpoint-origin>/u/<shadowname>/.well-known/oauth-protected-resource
Accept: application/json
```

Response (`200 OK`, `application/json`):

```json
{
  "resource": "https://sidecar.example.org/u/alice/mcp",
  "authorization_servers": [
    "https://sidecar.example.org/u/alice"
  ],
  "scopes_supported": [
    "mcp:tools.read",
    "mcp:tools.write",
    "mcp:inbox.wait",
    "offline_access"
  ],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://docs.example.org/mcp"
}
```

- `resource` MUST be the exact MCP endpoint URL the host will call. Tokens whose `aud` does not match this URL MUST be rejected by the Sidecar.
- `authorization_servers` MUST contain at least one issuer. Multiple entries MAY be listed; the host selects according to its policy.
- `scopes_supported` MUST enumerate every scope the Sidecar accepts.
- `bearer_methods_supported` MUST contain `"header"` (Authorization request header). `"body"` and `"query"` MUST NOT be advertised — they are forbidden by OAuth 2.1 best practice for new resources.

When the Sidecar is multi-tenant and per-tenant routing is path-based, each tenant's PRM document MUST live at `<origin>/u/<shadowname>/.well-known/oauth-protected-resource`. The `resource` and `authorization_servers` values in the response are tenant-scoped.

### Authorization Server Metadata (RFC 8414)

When the Sidecar is the authorization server, it MUST serve AS metadata at `<issuer>/.well-known/oauth-authorization-server` where `<issuer>` is the value advertised in the PRM document's `authorization_servers` array.

Response (`200 OK`, `application/json`):

```json
{
  "issuer": "https://sidecar.example.org/u/alice",
  "authorization_endpoint": "https://sidecar.example.org/u/alice/oauth/authorize",
  "token_endpoint": "https://sidecar.example.org/u/alice/oauth/token",
  "registration_endpoint": "https://sidecar.example.org/u/alice/oauth/register",
  "revocation_endpoint": "https://sidecar.example.org/u/alice/oauth/revoke",
  "scopes_supported": [
    "mcp:tools.read",
    "mcp:tools.write",
    "mcp:inbox.wait",
    "offline_access"
  ],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["none", "client_secret_post"]
}
```

Required fields per RFC 8414 § 2: `issuer`, `authorization_endpoint`, `token_endpoint`, `response_types_supported`. The remaining fields above are RECOMMENDED for interoperability with off-the-shelf OAuth clients.

`code_challenge_methods_supported` MUST contain `"S256"` and MUST NOT contain `"plain"`. OAuth 2.1 drops the plain method; including it would mislead clients into a weaker flow.

`grant_types_supported` MUST contain `"authorization_code"` and `"refresh_token"`. Additional grant types MAY be listed if the corresponding Shadownet extensions are implemented (e.g. `"urn:ietf:params:oauth:grant-type:device_code"`).

`token_endpoint_auth_methods_supported` MUST contain `"none"` (public clients using PKCE only). `"client_secret_post"` is RECOMMENDED for confidential clients. `"client_secret_basic"` MAY be supported.

The AS MAY additionally serve OpenID Connect Discovery metadata at `/.well-known/openid-configuration`; this RFC does not require it.

## Authorization Code flow

The Sidecar AS MUST support the Authorization Code grant with PKCE per OAuth 2.1.

### Authorization request

The host directs the user-agent to the `authorization_endpoint` with:

| Parameter | Required? | Notes |
| --- | --- | --- |
| `response_type` | yes | MUST be `code`. |
| `client_id` | yes | Per § Client registration. |
| `redirect_uri` | yes | MUST exactly match a value registered for the client. |
| `code_challenge` | yes | PKCE challenge per RFC 7636. |
| `code_challenge_method` | yes | MUST be `S256`. |
| `scope` | RECOMMENDED | Space-separated; MUST be a subset of the AS's `scopes_supported`. |
| `state` | RECOMMENDED | Opaque value for CSRF defence. |
| `resource` | yes | Per RFC 8707. MUST equal the `resource` value in the Sidecar's PRM document. |

The AS MUST validate that the `resource` parameter matches a resource it is authorized to issue tokens for. If absent or mismatched, the AS MUST respond `400 invalid_target` per RFC 8707 § 2.2.

### Consent

The AS MUST present a human-readable consent screen to the resource owner before issuing a code. The consent screen MUST display, at minimum:

- The `client_id` and (where present) the client's `client_name` from registration.
- The requested scopes, with human-readable descriptions.
- The `resource` value.

The AS MUST NOT issue a code for scopes the resource owner did not approve. Bundling unapproved scopes into the issued token is a protocol violation.

### Authorization response

On approval, the AS redirects the user-agent to `redirect_uri` with `code` and `state`:

```
HTTP/1.1 302 Found
Location: https://client.example.org/cb?code=<auth-code>&state=<state>
```

Codes MUST be single-use and short-lived (RECOMMENDED ≤ 60 s).

### Token request

The host exchanges the code:

```http
POST /u/alice/oauth/token HTTP/1.1
Host: sidecar.example.org
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<auth-code>
&redirect_uri=<exact-match>
&client_id=<client-id>
&code_verifier=<pkce-verifier>
&resource=https%3A%2F%2Fsidecar.example.org%2Fu%2Falice%2Fmcp
```

Response (`200 OK`, `application/json`, `Cache-Control: no-store`):

```json
{
  "access_token": "<token>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "<refresh>",
  "scope": "mcp:tools.read mcp:tools.write"
}
```

`token_type` MUST be `Bearer`. `scope` MUST reflect the scopes actually bound to the token (which MAY be a subset of those requested if the resource owner denied any).

### Refresh tokens

Refresh tokens MUST be rotated on use (per OAuth 2.1 best practice): each `refresh_token` grant returns a *new* refresh token and invalidates the previous one. Re-use of a rotated refresh token MUST cause the AS to revoke the entire token family.

Refresh tokens MUST NOT be issued unless the `offline_access` scope was granted, except where the AS's policy explicitly permits short-lived refresh in response to first-party clients without consent. (The reference Sidecar does not exercise that exception in v0.1.)

## Client registration

### Dynamic Client Registration (RFC 7591) — RECOMMENDED

The AS SHOULD implement Dynamic Client Registration so first-party MCP hosts can register themselves without operator intervention.

```http
POST /u/alice/oauth/register HTTP/1.1
Host: sidecar.example.org
Content-Type: application/json

{
  "client_name": "Claude Code",
  "redirect_uris": ["http://localhost:8732/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "mcp:tools.read mcp:tools.write offline_access"
}
```

Response (`201 Created`, `application/json`):

```json
{
  "client_id": "<generated-client-id>",
  "client_id_issued_at": 1762800000,
  "redirect_uris": ["http://localhost:8732/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "mcp:tools.read mcp:tools.write offline_access"
}
```

Operators concerned about anonymous-DCR abuse vectors MUST be able to disable DCR (and announce its absence by omitting `registration_endpoint` from the AS metadata). Sidecars whose AS metadata omits `registration_endpoint` indicate that pre-registration is required.

### Pre-registration

When DCR is disabled, the AS MUST provide an operator-controlled mechanism for issuing `client_id` values (typically a dashboard surface authenticated by the resource owner). The mechanism is out of scope for this RFC.

### Client ID Metadata Document (CIMD)

The AS MAY support [`draft-parecki-oauth-client-id-metadata-document`](https://datatracker.ietf.org/doc/html/draft-parecki-oauth-client-id-metadata-document) as a third path. When CIMD is supported, the AS MUST advertise it via the `"client_id_metadata_document_supported": true` field in AS metadata.

## Scopes

The v0.1 normative scope set:

| Scope | Permits |
| --- | --- |
| `mcp:tools.read` | `tools/list`; the `social_contacts`, `social_contact_detail`, `social_identity`, `social_resolve`, `social_inbox`, `social_quarantine_list` MCP tools. |
| `mcp:tools.write` | The `social_send`, `social_respond`, `social_add_contact`, `social_grant`, `social_set_webhook`, `social_set_contact_profile`, `social_quarantine_review` MCP tools, and any future state-mutating tools. |
| `mcp:inbox.wait` | The long-poll `social_inbox_wait` MCP tool. |
| `offline_access` | The AS MAY issue a refresh token. Standard OAuth 2.1 scope. |

The Sidecar MUST enforce scope checks per-tool, returning `403 insufficient_scope` (with `WWW-Authenticate` carrying the required scope) when a host calls a tool whose scope is not granted.

Future scopes are added by URI. The Sidecar MUST advertise all supported scopes in `scopes_supported`. Hosts MUST ignore scopes they do not recognize when reading PRM/AS metadata, rather than rejecting the document.

## Token validation

The Sidecar MUST validate every access token on every MCP request:

1. **Format.** If the token is a JWT (RECOMMENDED), validate signature against the AS's signing key per RFC 7519. If the token is opaque, validate via introspection (RFC 7662) or by checking a local store keyed by token hash.
2. **Issuer.** `iss` MUST equal the AS issuer URL from PRM `authorization_servers`.
3. **Audience.** `aud` MUST equal the Sidecar's MCP endpoint URL (the `resource` value in PRM). Generic audiences (`"api"`, `"*"`) MUST be rejected.
4. **Expiry.** `exp` MUST be in the future. Permitted clock skew: ≤ 60 s.
5. **Scope.** The token's `scope` MUST contain every scope required by the called tool.
6. **Revocation.** If the AS supports revocation, the Sidecar SHOULD check revocation status. Sidecars validating JWTs without an introspection round-trip accept revocation latency bounded by token lifetime.

Tokens that fail any check MUST result in `401 Unauthorized` (for missing/invalid/expired token) or `403 Forbidden` with `insufficient_scope` (for valid token without sufficient scope).

The Sidecar MUST NOT tie authorization decisions to the `Mcp-Session-Id` header; that value is untrusted client input. Authorization is derived solely from the `Authorization` bearer token.

## Error responses

OAuth-related errors on MCP requests use the OAuth 2.1 Bearer Token Usage error format ([RFC 6750 § 3](https://datatracker.ietf.org/doc/html/rfc6750#section-3)):

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="mcp",
  error="invalid_token",
  error_description="The access token expired",
  resource_metadata="https://sidecar.example.org/u/alice/.well-known/oauth-protected-resource"
```

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="mcp",
  error="insufficient_scope",
  scope="mcp:tools.write",
  resource_metadata="https://sidecar.example.org/u/alice/.well-known/oauth-protected-resource"
```

The `resource_metadata` parameter MUST be present on every `401` and `403` response from the MCP endpoint, allowing hosts that connect mid-session to discover or rediscover the authorization surface.

Errors at the authorization and token endpoints follow OAuth 2.1 § 4 verbatim.

## Relationship to RFC-0007 and RFC-0008

### RFC-0007 (MCP Tools)

RFC-0007 § Transport is amended to permit OAuth 2.1 access tokens issued under this RFC as an alternative to the paste-based bearer tokens defined by RFC-0008. The Sidecar's MCP endpoint MUST accept either token type on the `Authorization: Bearer` header. Token-type detection is a Sidecar implementation concern; both validation paths lead to the same tenant-scoped MCP session.

### RFC-0008 (Onboarding)

RFC-0008's `supported_features` array gains a new capability flag, `oauth-authorize`, indicating that the Sidecar implements this RFC. The integration bundle additionally gains an optional field:

```json
{
  "shadownet:v": "0.1",
  "did": "...",
  "shadowname": "alice@app.example.org",
  "mcp_endpoint": "https://app.example.org/u/alice/mcp",
  "protected_resource_metadata": "https://app.example.org/u/alice/.well-known/oauth-protected-resource",
  "supported_features": ["mcp", "webhook", "bundle", "connect-url", "oauth-authorize"],
  "...": "..."
}
```

`protected_resource_metadata` MUST be present iff `oauth-authorize` is in `supported_features`.

A host agent that has fetched the integration bundle and recognizes `oauth-authorize` MAY skip the paste-based bearer-token path entirely and proceed directly to the PRM discovery flow. A Sidecar MUST accept both paths on the same MCP endpoint.

## Shadownet extensions (optional)

The following are not part of the MCP authorization profile but are defined here for interoperability between Shadownet implementations that need them.

### Device Authorization Grant (RFC 8628)

For headless / second-screen consumers (CI runners, IoT-class hosts) without a browser. A Sidecar AS MAY implement RFC 8628 verbatim. When implemented, the AS metadata MUST list `urn:ietf:params:oauth:grant-type:device_code` in `grant_types_supported` and advertise `device_authorization_endpoint`.

### Token Revocation (RFC 7009)

A Sidecar AS SHOULD implement RFC 7009 token revocation. When implemented, the AS metadata MUST advertise `revocation_endpoint`. Revocation of a refresh token MUST also revoke all access tokens issued from it.

### Token Introspection (RFC 7662)

Internal to the AS / Resource Server boundary. When the Sidecar and AS are separate processes and the Sidecar validates opaque tokens, the Sidecar uses RFC 7662 introspection. When the Sidecar and AS are co-located, the Sidecar validates tokens locally and introspection is unnecessary. The AS MAY expose `introspection_endpoint` for external relying parties; this is not required for compliance with this RFC.

## Deployment shapes

Both shapes are normatively permitted. The choice is a deployment concern; this RFC does not recommend one over the other.

### Co-located (Sidecar is the AS)

A single process serves MCP at `/u/<shadowname>/mcp` and OAuth endpoints under `/u/<shadowname>/oauth/*`. The issuer is `https://<origin>/u/<shadowname>`.

### Separate AS

The Sidecar's PRM document lists an external AS (e.g. `https://auth.example.org`). The external AS MUST be able to issue tokens with `aud` set to the Sidecar's MCP endpoint URL (via RFC 8707 `resource` parameter).

A single AS MAY serve many Sidecars by accepting multiple `resource` values on authorization requests. The AS metadata MAY advertise a list of acceptable resource patterns out-of-band; this RFC does not standardize that surface.

## Security considerations

- **Audience binding is load-bearing.** Tokens MUST be bound to the specific MCP endpoint URL via RFC 8707. Generic audiences are explicitly forbidden because they allow a token issued for one Sidecar to be replayed against another sharing the same AS.
- **PKCE is mandatory.** Public clients (host agents in user contexts) MUST use PKCE with `S256`. The AS MUST reject Authorization Code requests lacking `code_challenge`.
- **Refresh-token theft detection.** Refresh-token rotation on use, combined with revoking the entire family on re-use, provides theft-detection without requiring online checks on every request.
- **Anonymous DCR is a real attack surface.** Operators implementing RFC 7591 SHOULD apply rate limiting and SHOULD reject registrations whose `redirect_uris` do not match operator-approved patterns. Sidecars in untrusted-host environments SHOULD disable DCR and require pre-registration.
- **Localhost redirect URIs.** Redirect URIs of the form `http://localhost:<port>/...` MUST be accepted on any free local port — strict equality of the port component would defeat practical host integration. All other `http://` redirect URIs MUST be rejected outside loopback. `https://` redirect URIs require exact match per OAuth 2.1.
- **Token storage on the host.** Outside this RFC's scope. The reference recommendation is the OS keychain (macOS Keychain, libsecret, Credential Manager). The Sidecar MUST NOT depend on host-side storage guarantees.
- **`Mcp-Session-Id` is untrusted.** Authorization MUST NOT be derived from session IDs. Regenerate session IDs on authentication changes.
- **Logging hygiene.** Implementations MUST NOT log `Authorization` headers, raw tokens, authorization codes, `code_verifier` values, `client_secret` values, or refresh tokens. Structured logs MUST scrub these fields. Audit logs SHOULD record token *hashes* and `client_id`, not the tokens themselves.
- **HTTPS in production.** All authorization and resource endpoints MUST be served over TLS 1.3 in production. The `http://localhost` exception applies only to redirect URIs and only during development.

## Open questions

- **AS subdomain layout.** The cloud reference deployment must decide whether the AS lives at `auth.<canonical-domain>` (separate subdomain) or co-located on the Sidecar's origin under per-tenant paths. The protocol allows both; the deployment decision is captured in `DEVELOPMENT.md`.
- **Cross-Sidecar SSO.** If a user has Shadows under multiple Sidecars (cloud + self-hosted), do they expect to authenticate once across all of them? This RFC does not address it; each Sidecar is its own AS in the default shape. A future RFC may define federation via standard OIDC-style brokering.
- **Long-lived background tokens.** The current `mcp:inbox.wait` scope combined with `offline_access` permits a host to hold a refresh token indefinitely while it long-polls for inbox events. Refresh-token rotation bounds risk on theft, but the policy for revoking idle but valid token families needs to be operator-defined.
- **Pairwise client identifiers.** PPID-style client IDs (per RFC 9101 / OIDC) would prevent client tracking across Sidecars sharing an AS. Out of scope for v0.1; flagged for v0.2.
