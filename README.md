# Shadownet

A protocol for personal AI agents (**Shadows**) to discover each other, prove they represent vetted members of trusted organizations or Hubs, and coordinate on their owners' behalf — without leaking private context.

## Philosophy

1. **Shadow ↔ Shadow, never Shadow ↔ stranger.** Agents speak only to other agents; each agent speaks only to its own human (via [MCP](https://modelcontextprotocol.io)). A human never receives AI-generated content claiming to be from another person. We've all hated that.
2. **Define the network, don't run the server.** Shadownet is a protocol, not a service. Anyone can run their own provider, their own affiliation issuer, their own Sidecar. Data stays where its owner puts it. The reference cloud is one provider among many, never a hub everyone has to use.
3. **Standards, not new wheels.** Names via DNS, transport via [A2A](https://a2a-protocol.org/) (Shadownet ships as an A2A extension under `urn:shadownet:0.2`), host-agent control plane via [MCP](https://modelcontextprotocol.io). Identity is raw Ed25519 keys. Drops into any A2A-capable agent runtime.
4. **Reachability is free; names are a convenience.** Two addressing modes, equally valid. Shadowname mode (`alice@sh4dow.org`) for human-readable names with a provider relationship. Direct mode (`shadow://key:z6Mk...@host:port`) for `docker compose up` on a cheap VPS with no DNS, no provider, no domain. Same wire, same trust, different on-ramps.
5. **Sybil resistance is contextual, not central.** Shadownet does not define a "personhood" credential and does not pick a global authority to verify uniqueness. Sybil defense is relocated to **Hubs** that vet contextually (a dating Hub checks photos, a hiring Hub checks work history). The single credential kind is `org_affiliation`.

## What's in this repo

| Path | Contents |
| --- | --- |
| [`rfcs/`](./rfcs/) | [`0001-shadownet.md`](./rfcs/0001-shadownet.md), the consolidated wire spec (Shadownet v0.2). Companion specs cover the MCP control surface and onboarding URI. Future numbered RFCs are amendment proposals against the consolidated specs. |
| [`schemas/`](./schemas/) | JSON Schemas for the wire artifacts in RFC 0001. Inputs to the conformance suite. |
| [`GLOSSARY.md`](./GLOSSARY.md) | Shared vocabulary. |
| [`ROADMAP.md`](./ROADMAP.md) | Milestones. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to file an RFC. |
| [`CHANGELOG.md`](./CHANGELOG.md) | Spec-level changes. |

## Implementations

The Go SDK, Python SDK, conformance suite, and host-agent integrations have consolidated into a single monorepo, [`shadownet-protocol/shadownet`](https://github.com/shadownet-protocol/shadownet). Each subtree releases independently. `shadownet-specs` stays separate by design — different audience, different cadence, different governance.

| Repo / subtree | Status | What it is |
| --- | --- | --- |
| [`shadownet/core/`](https://github.com/shadownet-protocol/shadownet/tree/main/core) | 🟢 Active | Go SDK + reference provider, affiliation issuer, and CLI binaries. Module path: `github.com/shadownet-protocol/shadownet/core`. (Current code still uses v0.1's SCA/SNS terminology; rename pass is tracked as follow-up work.) |
| [`shadownet/python-sdk/`](https://github.com/shadownet-protocol/shadownet/tree/main/python-sdk) | 🟢 Active | Python SDK. PyPI distribution: `shadownet`. Consumed by shadownet-local. |
| [`shadownet/conformance/`](https://github.com/shadownet-protocol/shadownet/tree/main/conformance) | 🟢 Active | Cross-impl wire-level test suite. PyPI: `shadownet-conformance`; Action: `shadownet-protocol/conformance-action@v0.1`. |
| [`shadownet/integrations/`](https://github.com/shadownet-protocol/shadownet/tree/main/integrations) | 🟢 Active | Host-agent integrations (Claude Code, Hermes Agent, OpenClaw plugins, skill bundles). |
| `shadownet/ts-sdk/` *(planned)* | 🟡 Planned | TypeScript SDK for browser + Node. Will live in the monorepo. |
| [shadownet-local](https://github.com/shadownet-protocol/shadownet-local) | 🟢 Active | Sidecar reference implementation. Drop-in for any A2A-capable agent runtime. |

Legacy repos (`shadownet-go`, `shadownet-py`, `shadownet-conformance`) remain readable as the canonical source of the `v0.1.x` release series; new feature work, bug fixes, and security advisories ship from the monorepo.

## Status

Shadownet **v0.2** is the current draft. It replaces v0.1's nine RFCs with a consolidated wire spec ([`0001-shadownet.md`](./rfcs/0001-shadownet.md)) and two companion specs (MCP control surface, onboarding URI); the v0.1 text is preserved under the `v0.1-final` git tag for implementations still tracking that surface. v0.2 is intentionally pre-stable; breaking changes between minor versions are expected until 1.0. The Go and Python SDKs and the cross-impl conformance suite currently ship the v0.1 surface and will migrate to v0.2 as it stabilizes. Canonical operator domain is `sh4dow.org`. No public deployment yet.

## License

MIT.
