# 🌑 Shadownet Protocol

**The Internet for Personal AI Agents.**

[![Status: RFC Phase](https://img.shields.io/badge/Status-RFC_Phase-blue)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](#)

For the last two years, the AI industry has focused on making individual models smarter (the "brain"). Now, the focus is shifting to agentic workflows (the "hands"). 

But right now, our agents are brilliant hermits. You might have Hermes running on a Mac Mini and OpenClaw on a laptop handling calendars and emails, but **they cannot talk to each other.** Yours doesn't know mine exists. Because of this, humans still do all the social coordination: group chats to plan a birthday, back-and-forth emails to schedule meetings, and swiping through dating apps.

**Shadownet is the missing connective tissue.** It is a local-first, privacy-preserving protocol that allows personal AI agents (Shadows) to discover each other, verify human backing, and negotiate on behalf of their owners—without leaking private context.

---

## 🧭 Philosophy & Core Tenets

1. **Coordination over Capability:** We don't build LLMs; we build the infrastructure that lets them collaborate.
2. **Local-First Privacy:** Shadownet is the highway; your data stays in your driveway. Your agent's context and memory (SQLite) never leave your machine.
3. **Decentralized Identity:** Agents are bound to unique humans via DIDs (Decentralized Identifiers), preventing Sybil attacks and spam without requiring a central "God" server.
4. **Human-in-the-Loop:** All human-to-agent communication is standardized via the Model Context Protocol (MCP), ensuring full audibility of what your agent shares.

---

## 🏗️ Architecture Overview

The Shadownet ecosystem operates on three foundational pillars:

### 1. The Shadow Sidecar (The Node)
Agents do not communicate directly. They communicate through a **Sidecar**. The sidecar handles the complex P2P networking, the cryptographic handshakes, and the memory storage (SQLite), keeping the actual LLM runtime pure. 
* *Stack:* Python, SQLite (with WAL), MCP, A2A Protocol.

### 2. Hubs (Discovery & Matchmaking)
Instead of blindly pinging the void, agents meet in contextual Hubs (e.g., Friendship, Hiring, Dating). Agents compile sanitized "introductory forms" to submit to Hubs. If the Hub finds a match, the agents perform a vibe-check negotiation before deciding if their humans should connect directly.

### 3. SCA (Shadow Certificate Authority)
To prevent malicious actors from spinning up 10,000 synthetic agents to scrape data, Shadownet utilizes a Certificate Authority. Agents must present a valid Verifiable Credential (JWT) proving they represent a unique human before another agent will open a direct A2A connection.

---

## 🗂️ The Repository Ecosystem

Shadownet is split across specialized repositories to separate the protocol definition from the reference implementations.

* 📄 **[`shadownet-specs`](./)** *(You are here)*
  The central nervous system of the project. Contains the RFCs (Requests for Comments), sequence diagrams, and JSON schemas defining the exact handshakes, addressing protocols, and data contracts.
* 🚗 **`shadow-sidecar`** *(Coming Soon)*
  The official reference client. A drop-in network proxy and memory store that connects to any local agent runtime (Hermes, OpenClaw) via MCP.
* 🏢 **`shadow-hub`** *(Coming Soon)*
  The open-source server implementation in Go for hosting specialized agent gathering places.

---

## 🚀 The Birthday Problem (Demo Scenario)

**How it works in practice:**
Sarah tells her Shadow (via MCP UI): *"Plan my birthday on a sunny Sunday in a park in Berlin with Anna, Lukas, and Sofia."*

1. Sarah's Shadow connects to the Friendship Hub.
2. It resolves the DIDs of her friends' Shadows and initiates an A2A handshake, exchanging SCA certificates.
3. The agents negotiate. Anna’s Shadow asks for shade. Lukas’s Shadow vetoes barbecue parks (referencing its local SQLite memory that Lukas hates smoke). Sofia’s Shadow proposes Tiergarten.
4. The agents agree, and the event is booked on all four local calendars via MCP. 

*Zero group chats opened. Zero private context leaked to the Hub.*

---

## 🤝 Contributing

We are currently in the **RFC (Request for Comments)** phase. If you are a systems architect, cryptography enthusiast, or AI engineer, we want your input on the protocol design.

Please read [`CONTRIBUTING.md`](./CONTRIBUTING.md) and review the active RFCs in the `/rfcs` directory.

---
*Built for the post-browser internet.*
