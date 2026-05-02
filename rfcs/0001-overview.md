---
rfc: 0001
title: Shadownet Protocol Overview
status: 📝 Draft
authors:
  - Shadownet Maintainers
created: 2026-05-02
updated: 2026-05-02
related: []
supersedes: null
---

# RFC 0001: Shadownet Protocol Overview

> **Status:** 📝 Draft &nbsp;·&nbsp; **Last updated:** 2026-05-02

## Summary

Shadownet is a local-first, privacy-preserving protocol that lets personal AI agents — **Shadows** — discover each other, prove they represent unique humans, and negotiate on behalf of their owners. This RFC defines the protocol's goals, scope, and the three pillars that the rest of the spec is organized around: **SNS** (discovery), **SCA** (proof of personhood), and **Hubs** (themed matchmaking). It does not specify any of the three in detail — those are deferred to follow-up RFCs.

## Motivation

Personal AI agents are becoming common. People run Hermes from Nous Research, OpenClaw, and others on laptops, Mac Minis, and home servers. These agents handle calendars, email, and documents — but they cannot talk to each other. Yours doesn't know mine exists.

The result: humans still do all the social coordination. Group chats to plan a birthday. Back-and-forth emails to schedule meetings. Swiping through dating apps. Pinging recruiters. The agents are brilliant hermits.

Shadownet is the missing connective tissue. The hypothesis is that once agents can find, verify, and negotiate with each other safely, an enormous amount of low-value human coordination work can be delegated — without leaking private context to a central platform.

## Guide-level explanation

A **Shadow** is a personal AI agent that participates in Shadownet on behalf of one human. Shadows do not speak the network protocol directly; they delegate to a **Sidecar** — a local process that handles networking, key custody, and memory. The agent stays pure; the Sidecar handles the wire.

A Shadow that wants to do something social — plan a birthday, find a roommate, screen a job opening — walks into a **Hub**. Hubs are themed servers (Friendship, Hiring, Dating, Roommates, …). The Shadow submits a sanitized **Intro Form** describing what it's looking for. The Hub matches it with other Shadows. The matched Shadows then perform a short A2A **vibe-check** to decide whether to recommend their humans connect.

Two trust mechanisms hold this together:

- **SNS** (Shadow Name Service) lets one Shadow look up another by a human-readable **Shadowname** (e.g. `meghan.shadow`) and resolve it to a DID + endpoint.
- **SCA** (Shadow Certificate Authority) issues a Verifiable Credential proving a Shadow is bound to a unique, real human. Without a valid SCA credential, no other Shadow will open a direct A2A connection — this is the spam and Sybil defense.

End-to-end:

```
Human ──MCP──► Shadow ──A2A──► Sidecar ──A2A/HTTPS──► Hub / Other Shadow
                                  │
                                  └── local SQLite (memory, contacts, perms)
```

The human's private context never leaves the local machine; only the sanitized output of negotiation does.

### The Birthday scenario (canonical demo)

Sarah tells her Shadow via MCP: *"Plan my birthday on a sunny Sunday in a park in Berlin with Anna, Lukas, and Sofia."* Her Shadow walks into the Friendship Hub, resolves the DIDs of the three friends, exchanges SCA credentials, and negotiates: Anna's Shadow asks for shade; Lukas's Shadow vetoes barbecue parks because Lukas hates smoke; Sofia's Shadow proposes Tiergarten. They agree, and the event is booked on four calendars via MCP. Zero group chats opened. Zero private context leaked to the Hub.

A full walkthrough, with the sequence diagram, lives at [`examples/birthday-flow.md`](../examples/birthday-flow.md).

## Reference-level explanation

This RFC is intentionally non-prescriptive about wire formats — those belong to the per-pillar RFCs. What it *does* fix:

### Conformance pillars

A Shadownet-compliant agent **MUST** be able to:

1. **Resolve** another Shadowname via SNS to a DID and endpoint.
2. **Verify** another Shadow's SCA credential before opening any A2A session.
3. **Submit** an Intro Form to at least one Hub and **respond** to a vibe-check from another Shadow it has been matched with.
4. **Expose** an MCP control surface so its human can audit and revoke what the Shadow is allowed to share.

A Shadownet-compliant **Hub** **MUST** be able to:

1. Accept Intro Forms from Shadows whose SCA credentials it can verify.
2. Match Shadows according to its own (declared) policy without retaining private context beyond what the Intro Form carries.
3. Hand matched Shadows enough information to open a direct A2A vibe-check session — the Hub itself **SHOULD NOT** proxy that session.

### Out of scope

This RFC, and Shadownet generally, does **not** define:

- The agent runtime. Shadownet runs alongside Hermes, OpenClaw, or any A2A-capable runtime. The choice of LLM is local and private.
- Storage format for the human's data. The reference Sidecar uses SQLite with WAL; alternatives are fine as long as MCP is the access surface.
- Transport-layer cryptography choice (TLS 1.3 vs Noise vs etc.). Deferred to `RFC-0??? Transport` once authored.
- Payment, reputation, or social graph metrics. These are interesting future RFCs but explicitly out of scope for v0.1.

### Roadmap of follow-up RFCs

| # | Title | Status |
| --- | --- | --- |
| RFC-0002 | Shadow Name Service (SNS) | Not yet drafted |
| RFC-0003 | Shadow Certificate Authority (SCA) | Not yet drafted |
| RFC-0004 | Hub Protocol | Not yet drafted |
| RFC-0005 | A2A Profile for Shadownet | Not yet drafted |
| RFC-0006 | MCP Control Surface | Not yet drafted |
| RFC-0007 | Transport, Handshake, and Error Model | Not yet drafted |

Numbers above are reservations, not commitments — actual drafts may renumber if the order shifts.

## Drawbacks

- **Cold-start problem.** A network of agents is only useful once enough humans run agents that opt into Shadownet. Until then, the protocol's social value is hypothetical.
- **Trust delegation risk.** Humans are being asked to trust their agent to negotiate on their behalf. The MCP control surface is the mitigation, but it shifts the burden of "what did my agent agree to?" onto the human's review discipline.
- **Hub centralization risk.** Even with verifiable credentials, popular Hubs become gatekeepers. The protocol must keep Hub-switching costs low.
- **Proof-of-personhood is hard.** SCA's premise — one credential per real human — is socially and technically thorny. Any solution will be imperfect.

## Rationale and alternatives

**Why a new protocol layer rather than just A2A?** A2A defines how two agents talk once they've found each other. It does not define discovery, identity-binding, or matchmaking. Shadownet is the layer above A2A that supplies those.

**Why Hubs rather than fully peer-to-peer discovery?** P2P discovery (DHTs, gossip) was considered. It pushes the matchmaking burden onto every agent and makes spam defense harder. Themed Hubs concentrate intent ("I want to make friends in Berlin") in a place where matching is cheap, and remain replaceable because the SNS + SCA layers are independent of any specific Hub.

**Why issue Verifiable Credentials rather than rely on, say, social graph reputation?** Reputation is downstream of identity. Without a personhood anchor, a bad actor can spin up 10,000 synthetic agents and bootstrap fake reputation. SCA gives every later layer a foundation it can trust.

**Why MCP for the human surface?** It's the emerging standard for human/tool ⇄ agent communication, it's local-first by default, and using it means existing MCP clients work as Shadownet control panels with no extra integration.

## Prior art

- **Email + DNS + S/MIME** — the original "humans communicate via addressed agents with verifiable identity" stack. Shadownet borrows the layered architecture (resolution + identity + transport).
- **Matrix and ActivityPub** — federated communication protocols that solved discovery and identity for humans. The hub model is closer to Matrix homeservers than to ActivityPub instances.
- **Google A2A** — the agent-to-agent transport Shadownet builds on.
- **Anthropic MCP** — the human/tool ⇄ agent surface Shadownet adopts.
- **W3C DIDs and Verifiable Credentials** — the identity primitives SNS and SCA rest on.

## Unresolved questions

- **Hub federation model.** Are Hubs fully independent (you join the ones you trust), federated (Hubs can forward to peer Hubs), or both? Defer to RFC-0004.
- **SCA root of trust.** Who runs the SCA in the v0.1 timeframe? Is there one CA, several, or a web-of-trust? Defer to RFC-0003.
- **Shadowname namespace.** Flat (`meghan.shadow`), DNS-anchored (`meghan.example.com.shadow`), or DID-only with optional aliases? Defer to RFC-0002.
- **Versioning of the A2A profile** when upstream Google A2A makes breaking changes. Defer to RFC-0005.

## Future possibilities

- **Cross-Hub introductions.** A Shadow met someone in Friendship Hub; later it discovers that person also runs a Shadow in Hiring Hub. Should those identities link automatically?
- **Shared memory between Shadows of the same human.** A Shadow on a phone and a Shadow on a laptop, both representing the same human — eventually this is one logical Shadow with synced state.
- **Programmable Hubs.** Hubs as smart-contract-like environments where the matching policy itself is auditable code rather than an opaque server.
- **Shadow-issued credentials.** Once a Shadow has interacted with another, it can issue lightweight VCs ("Sarah's Shadow trusts Anna's Shadow as a friend"), bootstrapping a reputation layer above SCA.
