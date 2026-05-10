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
| [`examples/birthday-flow.md`](./examples/birthday-flow.md) | Wire-level walkthrough across mixed cloud + self-hosted deployments — typed Interaction Profile path. |
| [`examples/free-form-coordination.md`](./examples/free-form-coordination.md) | Companion walkthrough — default free-form `text` envelope, no `interaction` URI. |
| [`GLOSSARY.md`](./GLOSSARY.md) | Shared vocabulary. |
| [`ROADMAP.md`](./ROADMAP.md) | Milestones. |
| [`DEVELOPMENT.md`](./DEVELOPMENT.md) | Component breakdown, repo layout, language picks, release surface. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to file an RFC. |
| [`CHANGELOG.md`](./CHANGELOG.md) | Spec-level changes. |

## Implementations

The Go SDK, Python SDK, conformance suite, and host-agent integrations have consolidated into a single monorepo, [`shadownet-protocol/shadownet`](https://github.com/shadownet-protocol/shadownet). Each subtree releases independently. `shadownet-specs` stays separate by design — different audience, different cadence, different governance.

| Repo / subtree | Status | What it is |
| --- | --- | --- |
| [`shadownet/core/`](https://github.com/shadownet-protocol/shadownet/tree/main/core) | 🟢 Active | Go SDK + reference SCA, SNS, and CLI binaries. Module path: `github.com/shadownet-protocol/shadownet/core`. |
| [`shadownet/python-sdk/`](https://github.com/shadownet-protocol/shadownet/tree/main/python-sdk) | 🟢 Active | Python SDK. PyPI distribution: `shadownet`. Consumed by `hermes-social`. |
| [`shadownet/conformance/`](https://github.com/shadownet-protocol/shadownet/tree/main/conformance) | 🟢 Active | Cross-impl wire-level test suite. PyPI: `shadownet-conformance`; Action: `shadownet-protocol/conformance-action@v0.1`. |
| [`shadownet/integrations/`](https://github.com/shadownet-protocol/shadownet/tree/main/integrations) | 🟢 Active | Host-agent integrations (Claude Code, Hermes Agent, OpenClaw plugins, skill bundles). |
| `shadownet/ts-sdk/` *(planned)* | 🟡 Planned | TypeScript SDK for browser + Node. Will live in the monorepo. |
| [`hermes-social`](https://github.com/meghancampbel9/hermes-social) | 🟢 Active | Sidecar reference implementation. Drop-in for any A2A-capable agent runtime. |

Legacy repos (`shadownet-go`, `shadownet-py`, `shadownet-conformance`) remain readable as the canonical source of the `v0.1.x` release series; new feature work, bug fixes, and security advisories ship from the monorepo.

## Status

The v0.1 protocol is drafted across seven RFCs. The Go and Python SDKs both ship the full v0.1 surface; the cross-impl wire-level conformance suite ships too. Canonical domain is `sh4dow.org`. The TypeScript SDK is next. No public deployment yet.

## License

MIT.
