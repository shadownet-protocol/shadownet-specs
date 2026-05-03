# Shadownet

A protocol for personal AI agents (**Shadows**) to discover each other, prove they represent real humans, and coordinate on their owners' behalf — without leaking private context.

## Philosophy

1. **Shadow ↔ Shadow, never Shadow ↔ stranger.** Agents speak only to other agents; each agent speaks only to its own human (via [MCP](https://modelcontextprotocol.io)). A human never receives AI-generated content claiming to be from another person. We've all hated that.
2. **Define the network, don't run the server.** Shadownet is a protocol, not a service. Anyone can run their own SCA, their own SNS, their own Sidecar. Data stays where its owner puts it. The reference cloud is one provider among many, never a hub everyone has to use.
3. **Standards, not new wheels.** Identity via [W3C DIDs](https://www.w3.org/TR/did-core/), claims via [W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model/), agent transport via [Google A2A](https://google.github.io/A2A/), human control via [Anthropic MCP](https://modelcontextprotocol.io). Drops into any A2A-capable agent runtime — Hermes, OpenClaw, Claude agents, anything that speaks MCP.

## What's in this repo

| Path | Contents |
| --- | --- |
| [`rfcs/`](./rfcs/) | The seven v0.1 RFCs (Overview, Identity, Credentials, SCA, SNS, A2A Profile, MCP Tools). |
| [`schemas/`](./schemas/) | JSON Schemas referenced by the RFCs. |
| [`examples/birthday-flow.md`](./examples/birthday-flow.md) | Wire-level walkthrough across mixed cloud + self-hosted deployments. |
| [`GLOSSARY.md`](./GLOSSARY.md) | Shared vocabulary. |
| [`ROADMAP.md`](./ROADMAP.md) | Milestones. |
| [`DEVELOPMENT.md`](./DEVELOPMENT.md) | Component breakdown, repo layout, language picks, release surface. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to file an RFC. |
| [`CHANGELOG.md`](./CHANGELOG.md) | Spec-level changes. |

## Implementations

| Repo | Status | What it is |
| --- | --- | --- |
| [`shadownet-go`](https://github.com/shadownet-protocol/shadownet-go) | 🟢 Active | Go SDK + reference SCA, SNS, and CLI binaries. |
| [`shadownet-py`](https://github.com/shadownet-protocol/shadownet-py) | 🟢 Active | Python SDK; consumed by `hermes-social` and `shadownet-cloud`. |
| [`hermes-social`](https://github.com/meghancampbel9/hermes-social) | 🟢 Active | Sidecar reference implementation. Drop-in for any A2A-capable agent runtime. |
| [`shadownet-conformance`](https://github.com/shadownet-protocol/shadownet-conformance) | 🟢 Active | Cross-impl wire-level test suite. |
| `shadownet-ts` | 🟡 Planned | TypeScript SDK for browser + Node. |
| `shadownet-cloud` | 🟡 Planned | First-provider deployment: signup, addressing, hosted Sidecars. |

## Status

The v0.1 protocol is drafted across seven RFCs. The Go and Python SDKs both ship the full v0.1 surface and are ready for downstream use. Wire-level conformance is being bootstrapped. The TypeScript SDK and the first-provider cloud deployment are next. No public deployment yet — the canonical domain is still being chosen.

## License

MIT.
