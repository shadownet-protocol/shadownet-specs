# RFCs

This directory holds the Request-For-Comments documents that drive every change to the Shadownet protocol. Each RFC is a numbered markdown file. Together they form the decision trail for the spec — *why* the protocol looks the way it does, not just *what* it looks like.

For the current normative state of the protocol, see [`/spec`](../spec/). For how a proposal moves from idea to accepted spec, see [`GOVERNANCE.md`](../GOVERNANCE.md).

## Status legend

| Symbol | State |
| --- | --- |
| 📝 | Draft |
| 💬 | Discussion |
| ✅ | Accepted |
| 🛠 | Implemented |
| 🪦 | Superseded / Withdrawn |

## Index

| # | Title | Status |
| --- | --- | --- |
| [0000](./0000-template.md) | RFC Template | — |
| [0001](./0001-overview.md) | Shadownet Protocol Overview | 📝 Draft |

## Filing a new RFC

```bash
cp rfcs/0000-template.md rfcs/0XXX-my-idea.md
```

Pick the next free number, fill in the template, open a draft PR. See [`CONTRIBUTING.md`](../CONTRIBUTING.md) for the full workflow.
