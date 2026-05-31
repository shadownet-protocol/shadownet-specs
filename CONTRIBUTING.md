# Contributing

Spec only — reference implementations live in sibling repos.

## Layout

- `/rfcs` — consolidated specs and amendment proposals (decision trail)
- `/schemas` — JSON Schemas for wire artifacts

Diagrams live inline in the RFC they belong to (Mermaid; renders on GitHub). Worked examples live in spec appendices (e.g. Appendix B of RFC 0001).

## Filing an RFC

```
cp rfcs/0000-template.md rfcs/0XXX-my-idea.md
```

Open a draft PR; mark ready for review when it's ready.

## Style

- Markdown, GitHub-flavored.
- Mermaid for diagrams.
- JSON Schema draft 2020-12 for schemas.
- MUST/SHOULD/MAY per [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) inside `/rfcs`.
