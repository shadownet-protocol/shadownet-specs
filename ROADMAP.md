# Roadmap

## Now

- **Sidecar (hermes-social)** — A2A transport, Ed25519 identity, JWT-EdDSA mutual auth, contact graph, per-contact grants, SQLite history, MCP control surface. Single-tenant. Working.
- **v0.1 RFC set drafted** — see [`rfcs/`](./rfcs/).

## v0.1 — known-contact coordination

Goal: two humans on different machines can ask their Shadows to coordinate something (schedule, plan, decide) over A2A, with one name lookup and one credential check in the middle.

| Track | Action |
| --- | --- |
| Spec | Stabilize RFCs 0001–0007. |
| Code | Multi-tenant Sidecar (host N Shadows in one process, per-user storage isolation). |
| Code | Cloud service: signup → Shadowname allocation → SCA cert issuance → MCP endpoint. |
| Code | 2-Shadow scheduling tracer-bullet end-to-end. |
| Code | 4-Shadow Birthday demo. |
| Code | Self-hosted Sidecar registration with the cloud SNS. |
| Code | Enable WAL mode in hermes-social. |

## Later

| Track | Action |
| --- | --- |
| Spec | Hub Protocol RFC (anonymous stranger-matching). |
| Spec | Interaction Profile RFCs (scheduling, intro, …) — analogous to iCalendar over email. |
| Spec | Federation RFC (multi-SNS, multi-SCA discovery and cross-trust). |
| Spec | Credential Transparency log. |
| Code | Reference Hub server. |
| Code | Vector / semantic memory in the Sidecar. |
