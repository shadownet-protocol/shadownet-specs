---
rfc: 0000
title: <One-line title in title case>
status: 📝 Draft
authors:
  - <Your Name> <your@email>
created: YYYY-MM-DD
updated: YYYY-MM-DD
related: []        # other RFC numbers this depends on or refines
supersedes: null   # RFC number this replaces, if any
---

# RFC 0000: <Title>

> **Status:** 📝 Draft &nbsp;·&nbsp; **Last updated:** YYYY-MM-DD

## Summary

One paragraph. What is being proposed, in plain language. A reader should understand the *shape* of the change without scrolling further.

## Motivation

Why are we doing this? What problem does it solve? What does the world look like today *without* this RFC, and what's wrong with it? Concrete scenarios beat abstract reasoning.

## Guide-level explanation

Explain the proposal as if you were teaching it to a new contributor. Use examples. Introduce new terminology — and add definitions to [`GLOSSARY.md`](../GLOSSARY.md) in the same PR. If the proposal changes existing behavior, contrast before/after.

## Reference-level explanation

The technical core. Be precise enough that a reference implementation could be built from this section alone. Cover:

- Wire formats (link to schemas in [`/schemas`](../schemas/))
- State machines and sequence diagrams (link to [`/diagrams`](../diagrams/))
- Error model
- Security and privacy properties
- Backwards compatibility (what breaks, what's deprecated)
- Conformance requirements (use **MUST** / **SHOULD** / **MAY** per RFC 2119)

## Drawbacks

Why might we *not* do this? Cost, complexity, lock-in, security surface, contributor friction. Be honest — every reasonable proposal has drawbacks; pretending otherwise weakens the case.

## Rationale and alternatives

- Why this design over the obvious alternatives?
- What other designs were considered? Why were they rejected?
- What is the impact of *not* doing this?

## Prior art

Other protocols, papers, or implementations that have tackled the same problem. Link generously. Note where Shadownet diverges and why.

## Unresolved questions

What parts of the design are still open? List them explicitly so reviewers know where to focus. Resolving these is a precondition for moving from 📝 Draft to ✅ Accepted.

## Future possibilities

Natural extensions of this RFC that are out of scope today but worth flagging. Helps future authors see the trajectory.
